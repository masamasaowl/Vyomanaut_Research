# Vyomanaut Research Topic List — LTS Track

**Status:** Authoritative. **Track:** LTS.
**Supersedes:** `reading-list.md` (original), `reading-list-v2.md`, `reading-list-v3.md`,
`reading-list-v3.1-addendum.md`. Delete none of them — retag each `Superseded by reading-list.md`
and keep as provenance for topic numbering R-01 → R-72.

**Scope.** Nothing here applies to the demo, frozen at `demo-v1.0.0` under ADR-062. Demo-track ADRs
appear only as *evidence about a decision the LTS must take for itself*.

**Incorporates:** the ADR-001–058 interrogation (v2), the ADR-059–073 pass (v3), the five-session
Design Council of August 2026 (v3.1), and a new assumption-debt audit of all 75 ADR files (§2).

---

## §1 — What this list is for, and the rule that changed

The previous three lists ordered research by *rewrite risk*, then by *milestone gating*. Both were
right at the time. This one adds a third and more uncomfortable axis, because the council sessions
exposed a pattern: **the project has been resolving by deliberation what it could have resolved by
reading.** ADR-072 was decided on a security argument about `chunk_id` that ADR-073 falsified one
session later; a capability-systems textbook would have flagged the same error in a paragraph.
ADR-065 through ADR-068 cite `This review` as their entire research source and set four operational
thresholds between them. Nine ADRs carry parameters explicitly labelled *"starting value"* with
tuning deferred to telemetry that will not exist until after the parameter has already done its
damage.

A council is the right instrument for a genuine tradeoff between two defensible positions. It is the
wrong instrument for a question that has a known answer someone else already published. **This list
exists to move as many questions as possible out of the second category.**

### The research-first trigger

Before convening a council on any question, check it against this. If **any** row matches, the
question goes to literature first and to council second, with the reading as an input.

| Trigger | Example from this project |
| --- | --- |
| The output is a **numeric threshold** on a stochastic process | `t = 24 h` polling, `72 h` departure, `4 h` heartbeat, `50 ms` p99 background gate, `5 s` gossip window |
| The question is a **known named problem** in another field | Relay provisioning is a reorder-point problem; timeout selection is failure-detector QoS; charge splitting is cost allocation |
| The decision rests on a claim about a **primitive's security properties** | ADR-072's `chunk_id` unforgeability argument |
| A **normative standard** exists | Key rotation cadence (NIST SP 800-57); metric naming (OpenMetrics); accessibility (ADR-050 did this correctly) |
| The number is a **vendor default** being promoted to a design constant | libp2p's 128 relay reservations; Badger's `ValueThreshold` |

Councils remain correct for: contested tradeoffs with no dominant answer (F-34's fix, repair
topology), rulings on *this project's* internal inconsistencies (F-03, Q26-4), and anything where
the literature exists but disagrees with itself.

---

## §2 — Assumption debt: where the project decided without reading

An audit of all 75 ADR files. This is the section the list is organised to serve — every item below
maps to a topic in §5–§11 that did not exist in v2 or v3.

### 2.1 — Class A: parameters declared "starting values"

Nine ADRs, fourteen parameters. Every one defers calibration to post-launch telemetry, and every one
is load-bearing before that telemetry exists.

| Parameter | ADR | Deferred to | The literature that exists |
| --- | --- | --- | --- |
| `t = 24 h` audit/absence polling | 006 | Q06-3, "production telemetry" | Failure-detector QoS (**Domain U**) |
| Heartbeat interval `4 h` | 028 | "may be reduced to 2 h if Q20-1" | Same |
| Background task p99 gate `50 ms` | 025 | "tuned empirically after launch" | Same, plus admission control |
| `GossipHealthyWindow = 5 s` | 048 | "starting value, consistent with this project's own" | Same |
| Release multipliers `0.95 / 0.80 / 0.65` | 024 | "tuned empirically after V2 launch" | Contract theory / threshold incentives (**Domain V**) |
| Vetting held-earnings cap `50%` | 024 | "if retention is low, consider 75%" | Same |
| Repair-ETA sample size `20 jobs` | 037 | *"arbitrary but non-trivial"* | Sequential estimation (**Domain V**) |
| Audit-secret rotation `90 d` | 067 | "a starting value" | **NIST SP 800-57 Pt.1** gives cryptoperiod guidance directly |
| RocksDB rate limiter `10 MB/s` | 023 | Q23-1 | Measurement, not literature → LaunchGate |
| Badger `ValueThreshold`, bloom FP, cache | 046 | "starting values" | Vendor defaults → **Class C** |
| Autostart restart count / delay | 047 | "starting values, not tuned" | Vendor defaults → **Class C** |
| Relay alert at `70% / 85%` | 068 | — | Burn-rate alerting (**Domain O**, R-56) |
| Provision 4th relay before `N = 400` | 068 | — | Reorder point under lead time (**Domain O**, R-57) |

**The pattern is not laziness.** In every case the ADR names the parameter honestly and flags it. The
defect is architectural: *"tune after launch"* is not a plan when the parameter governs whether
launch survives, and nine independent deferrals become a systemic exposure that no single ADR review
catches.

### 2.2 — Class B: council used where literature was available

| Decision | What was deliberated | What should have been read |
| --- | --- | --- |
| **ADR-072** — drop `file_id` from the capability token | Whether `chunk_id`'s randomness made `file_id` redundant | Object-capability literature: a capability names *authority over an object*, never its payload. The error was structural and named decades ago (**Domain W**) |
| **ADR-073** — client-submitted content-hash `chunk_id` | Four options weighed on wire-format cost | Content-addressed storage under adversarial clients (**Domain H**) |
| **ADR-061** — flat per-shard charge split | Two council sessions, `M10 contributor audit` as research source | Cost allocation with churn (**Domain J**, R-63 — now conditional) |
| **ADR-065–068** — launch gates, metric grammar, alert bijection, relay provisioning | `Source: This review` on all four; zero external citations | SRE alerting, inventory theory, OpenMetrics spec (**Domain O**) |
| **ADR-066** — dimensionless metric class | Invented from first principles | The OpenMetrics/Prometheus convention already handles this case |

### 2.3 — Class C: vendor defaults promoted to design constants

ADR-068 flags its own instance and is the model: *"the 128 slots/node figure is libp2p Circuit Relay
v2's **default** reservation limit, not a measured capacity."* The same shape is unflagged in
ADR-046 (three Badger defaults) and ADR-023 (RocksDB rate limiter). A default is a vendor's guess at
a median deployment; treating it as a capacity constant means the system's ceiling is set by someone
who has never seen it.

**Rule proposed:** any numeric constant sourced from a dependency's default must be tagged
`[VENDOR-DEFAULT]` in the ADR and carry a LaunchGate measurement ID.

### 2.4 — Class D: numbers inherited from a mismatched regime

Already the subject of Domain D, restated here because the audit found it is wider than churn:
ADR-010's MTTF 180–380 d comes from Storj's *NAS operator* population; ADR-009's 100 KB/s background
budget from Blake & Rodrigues (2003); ADR-006/007's thresholds from Bolosky (2000). Three different
regimes, none of them "Indian consumer desktop, 2026."

### 2.5 — Class E: load-bearing claims with no evidence

| Claim | Where | Status |
| --- | --- | --- |
| The 20% ASN cap "keeps the [disclosure] threshold safe" | ADR-022 | **False** (F-34). Council 4 ruled a cap cannot repair a collusion threshold |
| The repair process "must be P2P — no central entity fetches and re-encodes" | ADR-004 | **Violated by the implementation** (F-LTS-08) |
| Providers "are paid for passing audits, not GB stored" | ADR-012 | **Contradicted by the code** (F-LTS-09) |
| Per-provider audits give *retrievability* | ADR-002 | **Wrong primitive** (Q66-1) — gives possession |

### 2.6 — The positive exemplar

`ADR-029-addendum-a-bootstrap-durability.md` is what this list is trying to make routine. ADR-029
fixed the upload gate at ≥56 providers / ≥5 ASNs on purely structural grounds and never checked it
against a durability model. The addendum went to Paper 63, evaluated Theorem 9 at `N = 56`, showed
the integral, hand-checked it (*"removing 41 is necessary and sufficient, deterministically. The
integral returns exactly that. ✓"*), and produced the number the gate had been missing. **Structural
assumption → named paper → substitution shown → hand check → result.** That is the target shape for
every Class A parameter below.

---

## §3 — How to search

### 3.1 — Per-database syntax, paste-ready

```
IEEE Xplore  ("Document Title":TERM_A) AND ("Abstract":TERM_B) AND ("Publication Year":2018-2026)
             supports NEAR/n — ("Abstract":repair NEAR/5 bandwidth)
ACM DL       Title:(TERM_A) AND Abstract:(TERM_B)   → then filter by Publication: Proceedings of ...
ScienceDirect  TITLE-ABSTR-KEY(TERM_A AND TERM_B) AND NOT TITLE-ABSTR-KEY(POLLUTANT)
             8-connector limit; split rather than truncate
USENIX       site:usenix.org "TERM_A" "TERM_B"  (open access, so a general index beats their search)
DBLP         author-first for closed subfields — scan a canonical author's full publication list
Standards    NIST CSRC · IETF datatracker · W3C TR — for Class A parameters with a normative answer
```

### 3.2 — Standing exclusion set

Append to every query in Domains A, B, C, E, G, K, P, U:

```
NOT (blockchain OR "smart contract" OR token OR incentive-layer)
NOT ("federated learning" OR "model aggregation")
NOT ("cloud-of-clouds" OR "vendor lock-in" OR "storage class" OR "tier pricing" OR "cost model")
NOT (FPGA OR GPU OR "P4 switch" OR SmartNIC)
NOT ("fault injection" AND (hardware OR "soft error" OR radiation))     ← Domain Q only
NOT ("intrusion detection" OR "network security")                       ← Domain O only
```

### 3.3 — The four-part filter

| Field | Purpose |
| --- | --- |
| **Accept if** | What the paper must contain |
| **Reject if** | What disqualifies it outright |
| **Adjacent, not this** | The near-miss class this query is known to attract. Screen for this *first* — disqualifying is faster than qualifying |
| **Substitution test** | An arithmetic check at Vyomanaut's real parameters, run **during triage**, with working shown in the triage note |

### 3.4 — Triage scorecard

Score each abstract 0–2 on five axes. **≥7 draft · 5–6 mine the introduction · ≤4 discard.**

| Axis | 0 | 1 | 2 |
| --- | --- | --- | --- |
| Parameter reach | stated only ≥4× from ours | within 4× | covers or brackets ours |
| Trust model | assumes a trusted coordinator | partially trusted | untrusted providers *and* operator |
| Evidence type | proposal only | simulation | implementation + measurement |
| Actionability | needs a codec/protocol swap | a parameter change | adoptable behind an existing interface |
| Corpus delta | duplicates a paper we have | overlaps one | new mechanism or measurement |

### 3.5 — Citation-chain rule

For closed subfields (secure regenerating codes, PoR-on-repair, transparency-log gossip, ramp secret
sharing, failure-detector QoS), forward-citation from the domain's anchor beats keyword search.
Anchor → Semantic Scholar "cited by" → filter 2015+ → titles only. Budget one hour.

### 3.6 — Budget and null results

Two hours per topic, then either a shortlist or a written *"nothing found, here is what I tried."*
**A recorded null result is a deliverable.** Triage in batches of ten; draft one at a time.

Tags: **[GAP]** no corpus coverage · **[STALE]** expired measurement basis · **[OPEN-Q]** read, council
question remains · **[DEBT]** exists to retire a §2 assumption · **[NEW]** first appears here.

---

## §4 — Band 0: drafting queue, no searching required

| Work | Topic | Why it needs no search |
| --- | --- | --- |
| **Chen & Curtmola, "Remote data integrity checking with server-side repair"** (J. Computer Security 25(6), 2017) | R-47 | Found and verified. Its premise *is* Q68-3: removing the owner from the repair loop |
| **Chen, Ammula & Curtmola, "Towards server-side repair for erasure coding-based distributed storage systems"** (CODASPY 2015) | R-47 | The erasure-coded case rather than network-coded — closer to RS(16,56) |
| **Chen, Curtmola, Ateniese & Burns** (CCSW 2010) | R-47 | Origin of the RDC-on-repair line; read for threat model |
| **Paper 37 (SHELBY) — re-derivation, not a draft** | Q66-2 | Already in the corpus. ADR-060's re-derivation gets `S ≥ V/2,909` against the old `99×`. One afternoon; highest value-per-hour item in this document |

Papers 62 (LESS) and 64 (Hitchhiker) are **removed from the queue** — Council 2 closed the
code-family question (§5, Domain C).

---

## §5 — Band 1: gates an LTS milestone

### Domain P — Confidentiality-preserving repair `[GAP]` — **first priority**

**ADRs:** 004, 019, 020, 022, 026, 055 · **Finding:** F-69 (structural result)

Council 2 established the result this domain now serves. There are exactly three candidate
repairers, and each is disqualified by a property of the current code family: a **provider** must
gather `k=16` and obtains the plaintext; the **operator** obtains it, breaks ADR-019's stated trust
model, and carries all repair egress on one host; the **owner** is offline by design. **No
assignment of the repair role under RS + AONT-RS leaves every party blind.** F-69 is not a defect to
be relocated — it is a proof that the code family changes or a party is explicitly trusted.

**Must-read**

| Work | Why |
| --- | --- |
| **Pawar, El Rouayheb & Ramchandran**, "Securing dynamic distributed storage systems against eavesdropping and adversarial attacks" (IEEE TIT 2011) | Secrecy capacity of regenerating codes. R-27's anchor — run the citation chain from here |
| **Shah, Rashmi, Kumar & Ramchandran**, "Distributed storage codes with repair-by-transfer…" (IEEE TIT 2012) | R-28 exactly: a helper contributes without any party assembling `k` |
| **Rawat, Koyluoglu, Silberstein & Vishwanath**, "Optimal locally repairable and secure codes…" (IEEE TIT 2014) | Secrecy and repair locality together, with storage cost stated |
| **Goparaju, El Rouayheb, Calderbank & Poor**, "Data secrecy in distributed storage systems under exact repair" (2013) | Same first author as Paper 22 — chain half-walked already |
| **Resch & Plank, AONT-RS** — *Paper 16, held* | Re-read its security section against a repair-path adversary **before** searching outward |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-27** | Secrecy capacity of regenerating codes | States secrecy as a function of `(n, k, d)` with the storage cost of the guarantee | Assumes a trusted dealer present at repair | Eavesdropper-**on-links** models. Our adversary is the **helper node** — a different, less-studied case |
| **R-28** | Repair without any party assembling `k` | A helper contributes a function of its shard; no single party holds a decodable set | Requires the repairer to hold `k` in any intermediate step | MSR bandwidth-optimality results. They minimise bytes moved, not who can decode |
| **R-29** | Threshold cryptography applied to repair | Reconstruction distributed so no participant sees plaintext, at stated round/latency cost | Interactive protocols needing all `n` online | MPC generally — the cost is orders out for a 256 KB shard |

```
R-27  IEEE  ("Abstract":"secure regenerating codes" OR "secrecy capacity") AND ("Abstract":"distributed storage")
      chain forward-cite Pawar et al. 2011, filter 2015+
R-28  IEEE  ("Document Title":repair) AND ("Abstract":"repair-by-transfer" OR "helper node") AND ("Abstract":secre*)
      ACM   Title:(repair) AND Abstract:("without reconstruction" OR "partial decoding")
R-29  ACM   Abstract:("threshold cryptography" OR "secret sharing") AND Abstract:(repair AND storage)
```

**Substitution tests.** Evaluate every secrecy result at `(n=56, k=16, d=55)`. A construction whose
overhead is stated as "ℓ symbols sacrificed" needs `ℓ` computed at those values and compared against
RS(16,56)'s existing 3.5× expansion. **New, from Council 2:** evaluate against a repair event that is
*rare* (post-`r0`-gate, per ADR-076), not constant. Rarity does not make disclosure acceptable, but
it changes what a partial mitigation is worth.

---

### Domain K — Decoupling confidentiality from reconstruction `[GAP]`

**ADRs:** 014, 019, 022, 029, 031 · **Findings:** F-34, F-67, F-LTS-10

Council 4 ruled that **a placement cap cannot repair a collusion threshold** — a cap constrains
honest placement; collusion happens after it. Cap-tuning raises the attacker's price and does not
restore the property. The Scale Advocate's substitution bounds the mitigation anyway:
`2 × ⌊56f⌋ < 16` → `⌊56f⌋ ≤ 7` → `f < 0.143`, requiring **≥ 8 independent ASNs**; and at `f = 0.143`
three colluding ASNs still reach 21 ≥ 16, so cap-tuning buys exactly one coalition size.

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-17** ⬆ | **ASN diversity and detection feasibility for an Indian consumer pool** | Reports the count of independent ASNs actually reachable by residential providers, **and** treats CGNAT: a carrier ASN may cover millions, so ASN diversity may overstate failure-domain *and* collusion-domain diversity at once | Enterprise/datacenter AS topology | AS-level path diversity for routing. Right data, wrong question — we need *holder* diversity |
| **R-30** | Privacy threshold above reconstruction threshold | Raises disclosure above `k` while leaving `k` as reconstruction, with cost as a multiple of 3.5× | Requires a trusted dealer per file | Standard `(t, n)` secret sharing where `t` *is* the reconstruction threshold — that is what we already have |
| **R-31** | Collusion-resistant placement under a diversity budget | Constrains which *sets* may hold a stripe, with the durability cost of the constraint | Assumes failure domains abundant relative to stripe width | Rack-aware placement in datacenters. Same math, and the regime (5 domains, 56 shards) is inverted |

**R-17 is promoted from Domain E and from Band 2**, and is now the **feasibility gate**: if the
answer is < 8 independent ASNs, cap-tuning is off the table entirely and R-30 is the only
instrument. It absorbs F-LTS-10 — real ASN detection is unimplemented and registrations supply
`demo_asn`, so the cap is currently enforced against provider-supplied data.

**Must-read:** **Krawczyk, "Secret Sharing Made Short"** (CRYPTO 1993) — the computational-secret-
sharing construction AONT-RS descends from, and the first statement of R-30's exact shape. Read
before any 2020s ramp-scheme paper. Then **Blakley & Meadows (1984)** and **Yamamoto (1986)** for
ramp schemes; **Cidon et al., Copysets** (USENIX ATC 2013) and **Tiered Replication** (ATC 2015) for
R-31 — they constrain which node sets may hold a stripe, arrived at from durability; the math
transfers and the objective changes.

```
R-17  measurement + TRAI/M-Lab/Ookla/RIPE Atlas; APNIC AS-population data for IN
      ACM  Abstract:("autonomous system" AND (residential OR broadband) AND diversity)
R-30  chain forward-cite Krawczyk 1993 → "computational secret sharing", "ramp scheme"
      IEEE ("Abstract":"ramp secret sharing" OR "computational secret sharing") AND ("Abstract":threshold)
R-31  ACM  Title:(placement) AND Abstract:(correlated AND (diversity OR "failure domain"))
```

---

### Domain A — Audit continuity across repair `[OPEN-Q]`

**ADRs:** 002, 004, 014, 015, 017, 030, 059, 060 · **Questions:** Q68-3, Q69-1

Council 1 declined to rule on Q68-3 for cause: ADR-059 rejected microservice-side tagging because it
*"contradicts ADR-021's pure-P2P repair model,"* and F-LTS-08 showed pure-P2P repair does not exist.
Council 2 then restored provider-side repair (ADR-076), which puts the microservice option back off
the table. Q68-3 reduces to a two-way choice — give the repairing provider the authenticator keys
(it can then forge proofs for the position it just wrote), or leave reconstructed shards unauditable
until the owner returns — and **neither is acceptable.**

R-01, R-02, R-04 retired (Papers 66–72). **R-49 closed** — Q68-2 resolved by arithmetic: 286,720
chunks / 1,000 files → ~2.87 chunks per file-audit → ×256 blocks → ×1,000 files = **733,952 PRF
evaluations per provider per day**; at 1,000 providers, 7.34×10⁸/day ÷ 86,400 = **~8,494
HMAC-SHA-256/s**, a fraction of one core. Note the second-order consequence: this removes the
principal cost objection to the BLS variant, so if Fortress fails, **BLS must be re-priced, not
rejected on inherited grounds.**

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-47** ⬆ | **Tag generation for reconstructed shards without the owner** | A non-key-holder produces authenticators a verifier accepts, **or** the trilemma's cost is proved explicitly. Must name who holds what during repair | Requires the owner online at repair — the assumption ADR-004 removes | Dynamic-update PDP (Paper 69): verifies an update *claimed by the key holder*. The gap is authority, not verification |
| **R-48** | Detection composition across a 56-prover stripe | Gives `P(>n−k provers simultaneously corrupt-and-undetected)` with a stated independence assumption | Per-prover bounds restated — we hold three papers' worth | Multi-replica PDP (MR-PDP): replicas are identical, our 56 shards are distinct. Wrong composition |

```
R-47  IEEE ("Document Title":repair) AND ("Abstract":"remote data checking" OR "provable data possession")
           AND ("Abstract":"server-side" OR delegat* OR "without the client")
      chain forward-cite Chen & Curtmola 2017, filter 2017+
R-48  IEEE ("Abstract":"proofs of retrievability") AND ("Abstract":"multiple servers" OR dispersal) AND ("Abstract":detection)
```

**Substitution tests.** R-47: the scheme must work when the repairing party holds `k=16` and the
verifier holds `(k_prf, α₁…α₆₄)` for a file whose bytes it has never seen. R-48: substitute `n=56`,
`k=16`, tolerance 40, per-prover per-day miss probability `4.0 × 10⁻¹³` at `t=1%` (ADR-060's table);
if the bound assumes independence, name the common-mode cases it excludes (shared daemon release,
shared storage-engine bug).

---

### Domain G — Transparency logs and public randomness `[GAP]`

**ADRs:** 015, 017, 032, 059, 060 · **Findings:** F-22, F-23, F-68 · **Question:** Q70-1 (**resolved: yes**)

Council 1's deciding argument promoted this domain. ADR-060 has the microservice *"draw a fresh
32-byte challenge seed"* and **nowhere states the seed must be unpredictable to the verifier**. A
verifier free to choose seeds grinds until the derived 1% sample lands only on chunks a colluding
provider retained — defeating a 94.3%-detection scheme with no cryptographic break. **The beacon is
not Fortress's price; it is the fix, and Fortress arrives with it.**

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-50** ⬆ `[NEW]` | **Public randomness beacons** | Unpredictable before a deadline, publicly reconstructible after, verifiable by a party trusting nobody — with its liveness dependency stated | Requires a threshold of *staked* participants, or a chain for finality | VDFs alone: a VDF gives delay, not distribution. We need both; the paper must say which it provides |
| **R-23** | Gossip protocols for log consistency | Light clients detect a split view without a trusted monitor set | Requires every client to hold the full log | Byzantine broadcast generally — wrong cost class |
| **R-24** | Tamper-evident logging over mutable storage | Tamper-*evidence* where the substrate permits UPDATE (F-23/F-68 exactly) | Assumes append-only physical media | Blockchain anchoring — excluded by ADR-001 |
| **R-25** | Log-proof cost at fleet scale | Proof size and verification cost as a function of log length | Asymptotics only, no constants | Merkle-tree basics — held already |

**Must-read:** **Chuat, Szalachowski, Perrig, Laurie & Messeri** (CNS 2015) for R-23; **Crosby &
Wallach** (USENIX Sec 2009) for R-24's history-tree construction; **Meiklejohn et al., "Think Global,
Act Local"** (2020) — closest to our topology of many light clients and one operator; **Tomescu et
al.** (CCS 2019) for cost. Also the **Go checksum database** (`sum.golang.org`) and **Sigsum** —
deployed non-blockchain transparency logs with a gossip story, and LTS Session 19.0.1 already
*requires* verifying against one.

```
R-50  ACM  Title:("randomness beacon" OR "distributed randomness") AND Abstract:(unpredictab* AND verifiab*)
      IEEE ("Abstract":"public randomness") AND ("Abstract":bias-resistant OR unbiasable)
      also NIST Randomness Beacon spec · drand / League of Entropy (deployed, free to consume)
R-23  ACM  Title:(gossip) AND Abstract:("certificate transparency" OR "transparency log") AND Abstract:(consistency)
R-24  USENIX site:usenix.org "tamper-evident" logging history tree
```

**Substitution test (R-50).** ADR-060 draws one seed per `(provider, file, day)`; at 10⁶ files that
is 10⁶ seeds/day, which no beacon emits directly. The design must be **one beacon value per epoch,
expanded locally**, and any candidate must survive that expansion without losing unpredictability.
Show the expansion in the triage note.

**Price consumption before construction.** If consuming `drand` is acceptable, R-50 becomes an
integration question and most of this domain's cost disappears. Establish that before searching.

---

## §6 — Band 2: retires an assumption `[DEBT]` — **new band**

This band did not exist in v2 or v3. It is §2's remedy. Every topic here exists to replace a number
somebody guessed with a number somebody derived.

### Domain U — Failure detection and timeout selection `[GAP]` `[DEBT]` `[NEW]`

**ADRs:** 006, 007, 008, 025, 028, 048, 055 · **Retires:** five Class A parameters

Vyomanaut has **at least five independent timeouts** governing whether a node is considered present:
24 h polling, 72 h departure, 4 h heartbeat, 5 s gossip window, 50 ms background gate. All were set
by inspection of a bimodal absence histogram from 2000. None was derived. And there is a formal
literature on exactly this — how to select detector timeouts from a measured delay distribution to
hit a target detection time, mistake rate and query accuracy simultaneously — that the corpus does
not touch at all.

This is the largest single assumption cluster in the project and the cheapest to retire, because one
body of theory serves all five parameters.

**Must-read**

| Work | Why |
| --- | --- |
| **Chen, Toueg & Aguilera, "On the Quality of Service of Failure Detectors"** (IEEE Trans. Computers, 2002) | *The* paper. Defines detection time, mistake recurrence time and query accuracy probability as measurable QoS metrics, and gives a procedure for configuring a detector to meet them. It turns "24 h feels right" into a calculation |
| **Hayashibara, Défago, Yared & Katayama, "The φ Accrual Failure Detector"** (SRDS 2004) | Emits a suspicion *level* instead of a boolean, adapting to the observed delay distribution. Directly relevant to a bimodal population where one threshold cannot serve both nightly and weekend absence |
| **Défago et al., "The Accrual Failure Detector"** / later surveys | The maintained line, including deployed variants (Cassandra, Akka) with published tuning experience |
| **Bertier, Marin & Sens**, adaptive failure detection | Combines Chen's estimation with a dynamic safety margin — the shape needed when MTTF is itself unmeasured (Domain D) |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-65** `[NEW]` | **QoS-driven timeout selection** | Derives a timeout from a measured delay/absence distribution against stated detection-time and mistake-rate targets | Assumes a synchronous or bounded-delay model | Consensus liveness results. They assume a detector; we are choosing one |
| **R-66** `[NEW]` | **Accrual / adaptive detection under bimodal absence** | Handles a multi-modal or heavy-tailed absence distribution without a single hard threshold | Requires sub-second heartbeats — our cadence is hours | Datacenter failure detection at millisecond scale. Right theory, four orders out; read for method, not parameters |
| **R-67** `[NEW]` | **Detector cost at fleet scale** | Reports network and CPU cost per monitored node, and how it scales with fleet size and target accuracy | Full all-to-all monitoring | Gossip membership (Domain G / ADR-048). Related; that is dissemination, this is *decision* |

```
R-65  IEEE ("Document Title":"failure detector") AND ("Abstract":"quality of service" OR "detection time")
      chain forward-cite Chen/Toueg/Aguilera 2002 — this subfield is small and citation-chains well
R-66  ACM  Title:(accrual OR adaptive) AND Abstract:("failure detector") AND Abstract:(distribution)
R-67  IEEE ("Abstract":"failure detection") AND ("Abstract":scalab* AND (overhead OR cost))
```

**Substitution tests**

- **R-65:** substitute Bolosky's bimodal absence (nightly µ=14 h, weekend µ=64 h) and derive the
  timeout that meets a stated mistake rate. If the result is not near 72 h, ADR-007's threshold is
  wrong and the derivation says by how much. **This is the test that retires the parameter.**
- **R-66:** the population is explicitly bimodal and ADR-004 handles it with a single hard threshold
  plus a scheduler-priority workaround. Ask whether an accrual detector removes the need for the
  workaround entirely.
- **R-67:** at 1,000 providers with a 4 h heartbeat that is 6,000 messages/day inbound — trivial.
  At 100,000 it is 600,000/day, still trivial. **So cost is not the constraint and accuracy is;**
  state that explicitly so nobody re-argues the interval on bandwidth grounds.

---

### Domain V — Incentive thresholds and estimator design `[GAP]` `[DEBT]` `[NEW]`

**ADRs:** 024, 037, 053, 054, 061 · **Retires:** three Class A parameters

ADR-024 sets release multipliers at 0.95 / 0.80 / 0.65 and a 50% vetting held-earnings cap, and says
plainly that wrong thresholds *"create perverse incentives (too lenient: no deterrence; too strict:
punishes legitimate hardware failures)."* That sentence describes a well-studied tradeoff. PeerTrust
(Paper 31) is correctly cited for the *pattern* — an adaptive window — but not for the *values*, and
no source is cited for the values at all. ADR-037's *"20 completed jobs — arbitrary but non-trivial"*
is a sample-size question with a textbook answer.

**Must-read**

| Work | Why |
| --- | --- |
| **Bolton & Dewatripont, *Contract Theory*** (ch. on moral hazard with imperfect monitoring) | The exact structure: an agent's effort is unobservable, a noisy signal is observed, and payment is conditioned on the signal. Tells you how threshold placement trades deterrence against wrongful punishment |
| **Holmström, "Moral Hazard and Observability"** (Bell J. Economics, 1979) | The informativeness principle — which signals *should* enter the payment rule at all. Directly relevant to whether an aggregate score is the right statistic (Council 3's targeted-deletion residual) |
| **Hoffman, Zage & Nita-Rotaru, "A Survey of Attack and Defense Techniques for Reputation Systems"** (ACM CSUR 2009) | Attack taxonomy against *time-windowed* scores. Vyomanaut's three-window design is the structure these exploit |
| **Wald, *Sequential Analysis*** / any SPRT treatment | R-69. "How many samples before I act" is a solved problem; ADR-037's 20 is a guess at it |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-68** `[NEW]` | **Threshold placement under noisy monitoring** | Derives a payment threshold from a measured signal distribution and stated false-positive/false-negative costs | Assumes the principal observes effort directly | Reputation-score *aggregation* (Paper 31, held). That is how to compute the score; this is where to cut it |
| **R-69** `[NEW]` | **Sample size before acting on an estimate** | Gives a stopping rule from target confidence and effect size | Fixed-n designs with no error statement | A/B-test sizing. Right family; ours is sequential and one-sided |
| **R-70** `[NEW]` | **Under-punishment of targeted defection under aggregate scoring** | Treats an agent that defects on one principal and cooperates with others, under a scoring rule that aggregates across principals | Assumes a single principal | Sybil resistance (Domain I). Different attack: one identity, selective behaviour |

```
R-68  SD   TITLE-ABSTR-KEY(("moral hazard" OR "imperfect monitoring") AND threshold AND (payment OR contract))
      ACM  Abstract:("reputation threshold") AND Abstract:("false positive" OR deterrence)
R-69  SD   TITLE-ABSTR-KEY(("sequential analysis" OR "stopping rule") AND ("sample size") AND estimat*)
R-70  ACM  Abstract:(reputation) AND Abstract:("selective" OR targeted OR discriminat*) AND Abstract:(attack)
```

**Substitution tests**

- **R-68:** substitute ADR-024's own tiers. A provider at score 0.94 loses 25% of a month's earnings;
  at 0.79 it loses 50%. Compute the score distribution a legitimate desktop with one weekend outage
  per month actually produces (ADR-008's three windows, Bolosky's absence model) and check which
  tier it lands in. **If an honest provider routinely lands below 0.95, the top threshold is
  mispriced** and the derivation says by how much.
- **R-70:** this is Council 3's recorded residual. Flat split + aggregate multiplier means deleting
  one owner's shards while serving everyone else is punished only in aggregate. Quantify: at 1,000
  files per provider, what score penalty does total deletion of one file produce?

---

### Domain O — Operational readiness `[GAP]` `[DEBT]`

**ADRs:** 065, 066, 067, 068 · **Questions:** Q20-1, Q-M18-4, Q-M18-5 · **Retires:** four parameters

Four ADRs, `Source: This review` on all of them, zero external citations, and between them they set
the alert thresholds, the metric grammar, the rotation cadence and the relay provisioning rule.
ADR-067's own framing is the argument for this domain: *"A runbook nobody is paged into is a
document, not a control."*

**Must-read**

| Work | Why |
| --- | --- |
| **Beyer et al., *SRE* ch. 4 & 6; *SRE Workbook* ch. 5** | Multi-window multi-burn-rate alerting is the derived answer to ADR-068's 70%/85% shape |
| **Ewaschuk, "My Philosophy on Alerting"** | Short, and the source of the every-page-must-be-actionable rule ADR-067 implements without citing |
| **NIST SP 800-57 Part 1, *Recommendation for Key Management*** | **Directly answers Q-M18-4.** Gives cryptoperiod guidance by key type and usage. ADR-067's 90 days is a guess at a number a normative standard already publishes |
| **Zipkin, *Foundations of Inventory Management*** — base-stock / `(s, S)` policies | ADR-068's relay problem is a **reorder-point** problem: demand is provider growth, lead time is days, stockout is a provider that cannot join |
| **Clinical alert-fatigue literature** (e.g. Ancker et al., *BMC MIDM*, 2017) | The only field with measured data on responder behaviour as alert precision falls |
| **OpenMetrics / Prometheus naming spec** | ADR-066 invented a dimensionless class the upstream convention already handles |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-56** | SLO / burn-rate alerting | Derives thresholds from an error budget and a target detection time; reports the precision/recall trade | ML anomaly detection — we need thresholds a human can defend in a runbook | APM vendor whitepapers. No methodology |
| **R-57** | Capacity provisioning under long procurement lead time | Sets a reorder point from demand **variance** and lead time, not a utilisation percentage | Autoscaling — the difficulty is that relay capacity does *not* autoscale | Datacenter capacity planning. Wrong lead time, wrong granularity |
| **R-58** | On-call and runbook efficacy | Measures time-to-mitigate against runbook presence/quality, or the cost of low-precision alerts | Post-mortem collections with no measurement | Post-mortem *culture*. Valuable, not a control |
| **R-71** `[NEW]` | **Cryptoperiod selection** | Derives a rotation interval from key type, exposure and compromise-detection latency | Vendor rotation defaults with no rationale | Key *distribution* protocols. Different question |

```
R-56  ACM  Abstract:("service level objective" OR "error budget") AND Abstract:(alert*)
R-57  SD   TITLE-ABSTR-KEY(("reorder point" OR "base stock") AND "lead time" AND capacity)
R-58  ACM  Abstract:("on-call" OR runbook OR "incident response") AND Abstract:(measure* OR empirical)
      also PubMed "alert fatigue" override rate
R-71  NIST CSRC SP 800-57 Pt.1 (normative — read before searching); then
      IEEE ("Abstract":cryptoperiod OR "key rotation") AND ("Abstract":interval OR cadence)
```

**Substitution tests**

- **R-56:** ADR-068 fires warning at 70% for 1 h, critical at 85% for 15 m. Compute the detection
  time each gives against §27.5's growth model and compare against the actual lead time to warm a
  relay. **If the warning does not precede the ceiling by more than the lead time, the threshold is
  decorative.**
- **R-57:** 3 relay nodes × 128 slots = 384; at 45% CGNAT this binds at N ≈ 570, and the stated rule
  is "provision the 4th before N = 400" — a ~70% reorder point with an unstated demand variance.
  Substitute a growth rate and a lead time and check whether 400 survives. **Note that 128 is a
  libp2p default, not a measured capacity (Class C).**
- **R-71:** substitute ADR-067's 90 days against IC §8's 24-hour overlap window and the actual
  compromise-detection latency for the secrets manager. If detection latency exceeds the overlap
  window, rotation cadence is the wrong control and the overlap is the parameter to change.

---

### Domain W — Authority, capabilities and object naming `[GAP]` `[DEBT]` `[NEW]`

**ADRs:** 036, 059, 072, 073 · **Findings:** F-LTS-01, F-LTS-02 · **Retires:** a Class B decision

This domain exists because of a specific, expensive mistake. ADR-072 removed `file_id` from the
capability-token signing input on the argument that `chunk_id` was *"fresh, microservice-generated
randomness … never reused across files."* ADR-073 established one session later that `chunk_id` is
`SHA-256(chunk_data)` — a client-computed content address. The decision survived; the argument did
not. Council 5's fix was to bind `segment_id` and `shard_index` into the signing input at zero wire
cost, restoring the property that **a capability names authority over an object, never its payload.**

That principle is forty years old and is the first thing an object-capability text says. Reading it
would have cost an afternoon; not reading it cost two ADRs, a council session and a false claim in
the record. The same conflation appears a third time in Q68-3, where the authenticator binds content
and position simultaneously — which is why this is a domain and not a footnote.

**Must-read**

| Work | Why |
| --- | --- |
| **Miller, Yee & Shapiro, "Capability Myths Demolished"** (2003) | The canonical short treatment of what a capability is and what it is not. Would have caught ADR-072's error on page one |
| **Dennis & Van Horn, "Programming Semantics for Multiprogrammed Computations"** (CACM 1966) | Origin of the capability concept; read for the object/authority separation |
| **Shapiro, Smith & Farber, "EROS: a fast capability system"** (SOSP 1999) | A real system's treatment of revocation and expiry — the questions the 8-byte expiry field raises next |
| **Harnik, Pinkas & Shulman-Peleg, "Side Channels in Cloud Services: Deduplication in Cloud Storage"** (IEEE S&P Mag. 2010) | R-59: what a client learns from a content-addressed namespace it can probe |
| **Bellare, Keelveedhi & Ristenpart, "Message-Locked Encryption and Secure Deduplication"** (Eurocrypt 2013) | Formal treatment of content-derived identifiers and which security properties survive |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-59** | **Adversarial submission into a content-addressed namespace** | Treats the identifier as client-supplied and server-unverifiable; states what an attacker gains by squatting an existing address | Assumes the server hashes the content itself — structurally impossible here | Hash-collision resistance. SHA-256 is fine; this is namespace *policy* |
| **R-72** `[NEW]` | **Capability expiry, revocation and delegation** | Treats revocation of an outstanding capability before expiry, and the cost of the revocation check | Assumes a central revocation list checked on every use — the provider is offline-tolerant | OAuth/JWT revocation. Right shape, wrong trust model: our issuer is the party being constrained |

```
R-59  ACM  Title:("content-addressed" OR "content addressable") AND Abstract:(adversar* OR malicious OR namespace)
R-72  ACM  Title:(capability) AND Abstract:(revocation AND (expiry OR delegation)) AND NOT Abstract:(OAuth)
      chain Miller/Yee/Shapiro 2003 → EROS → seL4 capability papers
```

**Substitution test.** Under ADR-073, `chunk_id` is SHA-256 over AONT-RS ciphertext with a fresh
per-segment random key, so identical plaintexts yield different `chunk_id`s and the convergent-
encryption attack family does not apply. **That is a load-bearing property currently held by
accident** — it follows from ADR-019, not from any statement in ADR-073. State it as an invariant.
Then check the reverse: if `K` repeats (Domain N, F-47), `chunk_id`s collide across files.
R-72's substitution: capability lifetime is 8 bytes of expiry with no revocation path; compute what
a stolen token can do inside one expiry window against a provider that never contacts the
microservice.

**R-60 closed by construction** — see the substitution above. One verification outstanding: confirm
the client generates a fresh AONT key per segment *per upload*, including on re-upload of an
identical file.

---

## §7 — Band 3: invalidates a load-bearing parameter

### Domain D — Churn, availability, lifetime `[STALE]` — highest leverage, unchanged

**ADRs:** 005, 006, 007, 009, 010, 047, 053, 057 · **Findings:** F-06, F-15, F-16, F-72

R-12 → R-15 stand as written in v2. This domain now has a second consumer: **Domain U's derivations
are only as good as the absence distribution fed into them.** R-65's substitution test needs a
current distribution, not Bolosky's. Sequence D's measurement before U's derivation, or U produces
a well-derived answer to a stale question.

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-12** | Modern consumer-desktop session and uptime distributions | Reports session length distributions for consumer hardware, 2018+ | Datacenter or always-on server populations | Mobile app session analytics. Different device, different power model |
| **R-13** | Churn in *incentivised* storage networks | Departure/retention for paid node operators | Volunteer networks only — Paper 20's 87.6%-under-8h is explicitly the unpaid case | General P2P churn (Gnutella lineage). Held, and superseded by the regime question |
| **R-14** | Survival analysis under right-censored observation | Method for MTTF from a truncated window with censored exits | Requires complete lifetimes | Reliability engineering for hardware AFR. Related; ours is behavioural exit, not device failure |
| **R-15** | Indian residential broadband availability | Uptime, outage duration, power-availability data for IN residential | Enterprise or metro-fibre only | Mobile-network coverage studies |

**Where to look:** ProbeLab/Nebula IPFS measurement reports (2023–2025) — the modern successor to
Paper 20's data, continuously updated; **Storj published node-churn statistics** for R-13's
incentivised population; **Klein & Moeschberger, *Survival Analysis: Techniques for Censored and
Truncated Data*** for R-14 — a textbook problem in another field, not a distributed-systems search;
TRAI, M-Lab and Ookla open datasets plus CEA/state-utility outage data for R-15.

**Substitution test.** The paper must report a *distribution* whose median session length can be
compared against the 72-hour departure threshold and the 24-hour poll. A mean is not enough — the
whole objection to Bolosky is that `σ = 1.9 h` on a nightly absence is implausible for consumer
hardware, and a mean would have hidden that.

---

### Domain E — Correlated failure and the bootstrap regime `[GAP]`

**ADRs:** 001, 003, 004, 005, 008, 014, 029, 055 · **Findings:** F-28, F-34 · **Question:** Q63-1

R-16, R-18, R-19 as written in v2. **R-17 has left this domain** for Domain K, where it is now a
feasibility gate rather than context.

**Must-read:** **Ford et al., "Availability in Globally Distributed Storage Systems"** (OSDI 2010) —
the reference measurement of correlated failure in a real fleet, with statistical models for
placement and redundancy choices, and absent from the corpus. **Haeberlen, Mislove & Druschel,
"Glacier"** (NSDI 2005) — a *decentralised* system designed explicitly against massive correlated
failure, the closest structural analogue to Vyomanaut in the literature, also absent. **Cidon et al.,
Copysets** — shared with Domain K; one read, two domains.

**Substitution test.** Run at the ADR-029 gate: **56 providers, 5 ASNs, 20% cap.** `5 × 20% = 100%`
— the cap is exactly saturated and any two ASNs is 40% of the network. Any durability model assuming
failure domains are abundant relative to stripe width describes a regime Vyomanaut is never in at
launch.

---

### Domain C — What survives of erasure optimisation `[CLOSED except two]`

Council 2 settled Q26-4 from the code and Q62-1 followed. **R-09, R-10, Q62-2 and Q64-1 are closed;
Papers 62 and 64 are not drafted; ADR-026 closes as *no V3 repair-bandwidth optimisation*.** Once
ADR-076's `r0` gate exists, every repair event reconstructs 32 fragments and the single-block
literature delivers 0%, exactly as ADR-004's own table states.

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-11** | Small-object erasure coding | Treats objects near or below one stripe width, with the metadata-overhead crossover stated | Assumes large sequential objects | Small-file problems in HDFS. Right instinct, wrong redundancy model |
| **R-64** | Redundancy adaptation from observed reliability | Varies `(k, n)` or placement per device population from measured behaviour, with the re-classification transition cost | Requires a homogeneous fleet with a known AFR curve | Hot/cold *access*-based tiering (Papers 59–61). Same shape, different signal |

**Must-read:** **Kadekodi, Rashmi & Ganger, "Cluster storage systems gotta have HeART"** (FAST 2019)
and **"Tiger: Disk-adaptive redundancy without placement restrictions"** for R-64.

**Correction still outstanding:** ADR-026's Clay rejection states `α ≥ 40^16`; the standard MSR bound
at `(56, 16, 55)` gives `40² = 1,600` — a 164-byte sub-chunk. Clay very likely stays ruled out on
I/O grounds, but recompute, show the substitution, and change the argument before the ADR closes.

---

## §8 — Band 4: structural stability

Unchanged from v3 in content. These three domains are about whether the implementation is true to
the design and whether the numbers published about it can be defended. ADR-070 found thirteen
defects by running one seam once; F-LTS-07 and F-LTS-08 are two more of the same class found by
reading code against documents. Nobody has counted how many seams there are.

### Domain Q — Deterministic simulation, fault injection, seam verification `[GAP]`

**Must-read:** **Yuan et al., "Simple Testing Can Prevent Most Critical Failures"** (OSDI 2014) —
read first; its finding that most catastrophic failures come from mishandled *non-fatal* errors is
ADR-070's thirteen findings stated as a general result. **Alvaro, Rosen & Hellerstein,
"Lineage-driven Fault Injection"** (SIGMOD 2015). **The FoundationDB paper** (SIGMOD 2021).
**Kingsbury & Alvaro, "Elle"** (VLDB 2020). **Leesatapornwongsa et al., "SAMC"** (OSDI 2014).

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-51** | Deterministic simulation of the real implementation | Runs the *real* code under a controlled scheduler and clock; states what it cannot make deterministic | Tests a *model* — ADR-069 rejects that for this reason | Property-based testing of one component |
| **R-52** | Fault injection targeted at seams | Chooses injection points from dataflow or lineage, not uniformly; reports bugs-found-per-run | Chaos-as-practice with no selection principle | Network-partition testing alone. ADR-070's findings were header, encoding and identity mismatches |
| **R-53** | Lightweight formal specification of lifecycle state machines | A specification checked *against the implementation*; reports sync cost | Full refinement proofs | Verifying consensus. Ours is `PENDING_ONBOARDING → VETTING → ACTIVE → DEPARTED` and who writes which column |

**Substitution test (R-52).** The thirteen `F-070-N` findings plus F-LTS-07/08 are the benchmark.
*Would this technique have found F-070-4 (a missing `Authorization` header) and F-070-11 (a cache TTL
with no refresh loop)?* If no to both, it is the wrong technique for this system.

**Cheapest item here is not literature.** ADR-070's rule — a milestone adding a stage to a
multi-stage lifecycle must be verified live against the stage before and after — should become a
machine-checked VERIFY-block requirement in the build skill. Do that before R-51.

### Domain S — Scale-claim validity `[GAP]`

**Must-read:** **Jansen, Tracey & Goldberg, "Once is Never Enough: Foundations for Sound Statistical
Inference in Tor Network Experimentation"** (USENIX Security 2021) — the best fit in this document;
substitute "sampled Tor network" for "three-tier synthetic population" and it is written about
ADR-069. **Shadow** (runs real application binaries). **Gouveia et al., "Kollaps"** (EuroSys 2020).

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-54** | Fidelity of mixed real/synthetic populations | Quantifies what a reduced-fidelity node fails to reproduce and bounds the induced error | Pure emulation-vs-hardware with no reduced tier | Digital twins — fidelity to a physical plant, not a protocol |
| **R-55** | Statistical inference from sampled network experiments | A procedure for confidence intervals and required run counts over a sampled network | Single-run evaluations — what it argues against | Microbenchmark variance. Right instinct, wrong scale |

**Also fix F-LTS-03 first** — ADR-069's synthetic tier can regenerate chunk bytes via
`PRF(node_seed ‖ chunk_id)` and so can compute `μ`, but cannot compute `σ` without ADR-059's
per-chunk authenticators. A synthetic tier that cannot answer the audit primitive the LTS ships is
not measuring the system.

---

## §9 — Band 5: cheap now, expensive later

### Domain L — Single-writer election `[GAP]`

R-26, R-32, R-33 as in v2. **R-32 is the high-value item**: if reservation-style bounded counters
work, coordinated operations drop below six and ADR-013's central trade improves materially.

**Must-read:** **Balegas et al., "Extending Eventually Consistent Cloud Databases for Enforcing
Numeric Invariants"** (SRDS 2015) and "Putting Consistency Back into Eventual Consistency"
(EuroSys 2015) — R-32 precisely: preserving a `≥ 0` floor by reservation rather than by coordinating
every operation; escrow debit is the textbook case. **Burrows, "The Chubby Lock Service"** (OSDI
2006) for lease semantics and fencing. **Kleppmann, "How to do distributed locking"** — the clearest
statement of why a lease without a fencing token is not a lock, which is F-35/F-76 exactly.

### Domain M — Local secrets `[GAP]`

R-34 → R-37 as in v2. **Must-read:** **Blocki, Harsha & Zhou, "On the Economics of Offline Password
Cracking"** (IEEE S&P 2018) — states R-34 in the currency the decision needs, guesses per rupee;
ADR-020's `t=3 / m=64 MB` is an *interactive* setting facing an *offline* attack. **Bonneau, "The
Science of Guessing"** (S&P 2012) for R-35's distribution. **Jarecki, Krawczyk & Xu, "OPAQUE"**
(Eurocrypt 2018) for R-36 — removes the oracle rather than raising its cost. Windows DPAPI, macOS
Keychain ACLs and libsecret documentation for R-37, checked against **no elevation** (ADR-042) and
**unattended start** (ADR-047/051).

**Price first, before four search topics:** do not store `pointer_ciphertext` server-side at all. It
trades ADR-020's Scenario 4 recovery for removing the entire oracle.

### Domain N — AEAD nonce handling and RNG failure `[GAP]`

R-38 stands. **R-39's urgency is reduced** — F-LTS-01 closes on ADR-072-A's position-binding fix, so
authorisation integrity no longer depends on AONT key freshness; the domain returns to
confidentiality only.

**Must-read:** **Heninger, Durumeric, Wustrow & Halderman, "Mining Your Ps and Qs"** (USENIX Sec
2012) — measured in-the-wild entropy failure producing duplicate keys, which turns "theoretically
fragile" into a probability. **Ristenpart & Yilek, "When Good Randomness Goes Bad"** (NDSS 2010) —
the VM-cloning and snapshot-restore case, exactly what `nonce = 0` is most exposed to. **Gueron &
Lindell, AES-GCM-SIV** (CCS 2015) / RFC 8452 and libsodium's XChaCha20-Poly1305 for R-38.

### Domain I — Coordination DoS, reputation gaming, relay `[GAP]`

R-40, R-41, R-42 as in v2. **R-42 is promoted within the domain** — ADR-068 makes relay capacity the
binding constraint. Note the split: R-42 asks *how much capacity is needed under burst*; R-57 asks
*when to buy it*. Different literatures; do not merge.

**Must-read:** **Hoffman, Zage & Nita-Rotaru** (ACM CSUR 2009) — shared with Domain V. **Juels &
Brainard, "Client Puzzles"** (NDSS 1999) for R-40; our clients are *registered*, which is a lever
most DoS work does not have.

---

## §10 — Band 6: economics, compliance, lifetime

### Domain J — Payments, Sybil bound, carbon `[WEAK]`

R-43 → R-46 as in v2. **R-44 is not academic literature** — RBI PA/PG guidelines and Razorpay's
escrow/Route documentation are the primary sources. **R-63 is conditional on a count, not a search:**
its trigger was ADR-061's mid-period handover problem, and under ADR-076 repair becomes rare, so
handovers become rarer still. **Count the handover fraction first; under ~1% it closes unstarted.**

**Must-read:** **Gupta et al., "Chasing Carbon"** (HPCA 2021) and **"ACT: Designing Sustainable
Computer Systems with an Architectural Carbon Modeling Tool"** (ISCA 2022) — ACT is a *tool*, which
is what a defensible per-terabyte figure requires. **Tannu & Nair, "The Dirty Secret of SSDs:
Embodied Carbon"** (HotCarbon 2022) — per-device embodied carbon for storage specifically, six
pages, and Q44-1's missing half. **CarbonClarity** (HPCA 2024) — estimates carry large uncertainty;
ADR-044's claim needs error bars or it will not survive a hostile reading.

**F-41 remains the sharpest item here** and is a document-consistency fix, not a search: ADR-024 §5
justifies seizure as cost recovery while §7 says the operator bears no repair cost.

### Domain T — Cryptographic agility and post-quantum sequencing `[GAP]`

**Question:** Q72-1. Keep this domain small deliberately — two topics and a design rule.

R-61 (crypto-agility as a protocol property) and R-62 (PQ signature cost on consumer hardware) as in
v3. **Must-read:** NIST FIPS 203/204/205 as primary; the pqm4 benchmark project for consumer-class
numbers.

**Substitution tests.** R-61: audit every wire format in IC §3.1, §4.1 and §4.3 for an algorithm
identifier. If none carries one, adding it later is a version bump — the same wall ADR-073's Option
(C) hit. **Do this audit at M19, when the interfaces are open anyway.** R-62: Ed25519 signatures are
64 bytes; ML-DSA-44 signatures are ~2.4 KB. Substitute into a 1,040-byte audit response and a
262,252-byte upload frame at ADR-060's audit volume and state the bandwidth delta.

**The deliverable is the design rule, not the papers.** If the M19 audit finds no algorithm
identifiers, the output is a one-line ADR adding a version byte to the signing-input domain prefixes
— the mechanism ADR-059 already uses (`"vyomanaut/por/prf/v1"`), just applied consistently.

---

## §11 — Not research

### 11.1 — Moved to LaunchGate measurements (ADR-065)

R-20 (EC library throughput → Q65-1), R-21 (symmetric crypto on low-end hardware → Q-M18-1), R-22's
measurement half, Argon2id parameterisation, RocksDB/Badger tuning (Q23-1, Q27-1, Q49-1), and the
three Class C vendor defaults. Each gets a `LaunchGate` entry and a
`scripts/benchmarks/results/{id}.{profile}.json` artifact. **Domain F is reduced to R-22's literature
half only.**

### 11.2 — Council or ruling, not reading

Only two remain of the seven in the previous list:

| Item | Why it is not research |
| --- | --- |
| **Q66-2** — SHELBY re-derivation | Paper 37 is in the corpus. Arithmetic with the working shown |
| **F-LTS-11** — where ADR-059's 4,096 B of authenticators live relative to Frame 1's fixed 262,252-byte layout and `chunk_id = SHA-256(chunk_data)` | A specification decision. Blocks Proof of Storage implementation |

Resolved by the August 2026 council and removed: F-01, Q68-1, Q68-2, Q66-1, Q26-4, Q62-1, Q62-2,
Q64-1, F-03, F-LTS-04, F-LTS-01, F-34's instrument question.

### 11.3 — Carried, unchanged priority

vLog GC fine/coarse reclaim (ADR-023) — a real divergence between `build.md` Session 5.1.5 and
ADR-023's prose; independent of everything above and runnable in parallel. Upload straggler / Storj
`o` (ADR-003) `[V3]`. Adaptive polling from score history (ADR-008) `[V3]` — now downstream of
**Domain U**, not just Domain D. Post-graduation monitoring of forced-ceiling providers (ADR-053,
Q57-2). ISP data-plan sync (ADR-056) `[V3]`. Background execution continuity (ADR-057, Q58-1).
Desktop shell cluster (ADR-038–052) — low because cheap to reverse, not because well-evidenced.
Hot/Cold band parameters (ADR-018) — **stop searching**; Papers 59–61 closed it and Q59-1 is a
product call. DHT necessity at small N (ADR-001) — **suspended**, subordinate to F-09/F-10.

---

## §12 — Sequence

```
IMMEDIATE — no reading, no council
  ├── ADR-076 (repair topology) · ADR-072-A (position binding) · ADR-012-A (payment unit)
  ├── ADR-022-A (remove the cap's confidentiality claim) · ADR-026 → Closed
  ├── ADR-077 (research-first triage; tags Class A/C parameters)
  ├── TestProfileConfidentialityMarginHolds — ten lines, fails on prod today, and should
  ├── Track: tags on ADR-059/060/061/065/066/067/068; demo ADR ceiling → 073
  └── Verify: fresh AONT key per segment per upload  → closes R-60

STILL OPEN, no reading required
  ├── Q66-2 — SHELBY re-derivation against Paper 37 (held)   ← one afternoon, best value/hour
  ├── F-LTS-11 — authenticator placement vs Frame 1          ← blocks Proof of Storage
  └── Count the mid-period handover fraction                 ← decides whether R-63 exists

M19 — LTS Foundation (no research precondition)
  └── R-61's substitution test: audit every wire format for an algorithm identifier

CONFIDENTIALITY  ← moved ahead of Proof of Storage
  R-27 → R-28 → R-29     P   R-28 now load-bearing
  R-17                   K   ASN feasibility + detection  ← gate on whether cap-tuning exists
  R-30 → R-31            K
  (ADR-076's repair-topology milestone runs alongside — P's substitution test is defined
   against a post-r0-gate repair frequency)

PROOF OF STORAGE
  R-47                   A   Band 0, no search       ← GATING
  R-50                   G   price drand first       ← HARD DEPENDENCY
  R-48                   A

ASSUMPTION RETIREMENT  ← new, runs in parallel from here; each topic retires a live parameter
  R-12 → R-15            D   ...then R-65 → R-67   U   (D before U, or U derives from stale data)
  R-68 → R-70            V
  R-56 → R-58 · R-71     O
  R-59 · R-72            W

THEN
  R-16 · R-18 · R-19  E │ R-51 → R-53  Q │ R-54 · R-55  S │ R-23 → R-25  G
  R-26 · R-32 · R-33  L │ R-34 → R-37  M │ R-38 (R-39↓)  N │ R-40 → R-42  I
  R-43 → R-46 (R-63?) J │ R-61 · R-62  T │ R-11 · R-64   C
```

**If only one domain gets attention: P.** Every repair event discloses plaintext to *someone*, by
construction, today, with no attacker — and the operator is currently that someone.

**If only two: P and A.** The coupling is now proven rather than asserted: Domain P and Q68-3 are the
same moment in the protocol seen from confidentiality and from auditability.

**If only three: add U.** This is a change. Domain U is the cheapest large win in the document —
one body of theory (Chen/Toueg/Aguilera, then accrual detectors) retires five guessed parameters
across seven ADRs, and unlike Domain D it does not wait on a measurement campaign to *start*.

---

## §13 — Running it

1. **Separate the queues.** "Research" is five activities competing for the same hours: rulings,
   derivations, search, drafting, measurement. §11.2 has two items needing no reading at all; they
   should not sit behind a search queue.
2. **Apply the §1 trigger before every council.** If a row matches, the reading is an *input* to the
   session, not an alternative to it.
3. **Cap search at two hours per topic.** A written null result is a deliverable.
4. **Triage in tens, draft in ones.** The §3.4 scorecard makes triage batchable and delegable.
5. **Model allocation.** Sonnet for scorecard triage and first-draft `paper-NN` structure; Opus for
   substitution tests, council sessions, and cross-document reads.
6. **Every artifact carries a `Track:` line.** Same rule as ADR-062 §3.

## §14 — The standing rule

**Any formula or parameter imported from a paper carries its substitution inline.** Four instances
now: F-43 (`40^16` vs `40²`), NFR-044 (an interpolation formula wrong at its own anchor), Q66-2
(`99×` vs `S ≥ V/2,909` — a factor of ~288,000 in the opposite direction, from a paper already
held), and the §2 audit's fourteen "starting values," which are the same failure with no paper at
all behind them.

§3.3's **substitution test** field moves that rule upstream from ADR-drafting to shortlisting, which
is the cheapest place it can live. **§2's assumption-debt audit is what happens when it is absent.**

---

## Appendix — Topic index

| Band | Domain | Topics |
| --- | --- | --- |
| 0 | Drafting queue | Chen & Curtmola ×3, Q66-2 |
| 1 | **P** Confidentiality-preserving repair | R-27, R-28, R-29 |
| 1 | **K** Threshold decoupling | R-17, R-30, R-31 |
| 1 | **A** Audit continuity | R-47, R-48 |
| 1 | **G** Transparency logs, randomness | R-23, R-24, R-25, R-50 |
| 2 | **U** Failure detection `[NEW]` | R-65, R-66, R-67 |
| 2 | **V** Incentive thresholds `[NEW]` | R-68, R-69, R-70 |
| 2 | **O** Operational readiness | R-56, R-57, R-58, R-71 |
| 2 | **W** Authority and capabilities `[NEW]` | R-59, R-72 |
| 3 | **D** Churn | R-12, R-13, R-14, R-15 |
| 3 | **E** Correlated failure | R-16, R-18, R-19 |
| 3 | **C** Erasure remnant | R-11, R-64 |
| 4 | **Q** Seam verification | R-51, R-52, R-53 |
| 4 | **S** Scale-claim validity | R-54, R-55 |
| 5 | **L** Single-writer election | R-26, R-32, R-33 |
| 5 | **M** Local secrets | R-34, R-35, R-36, R-37 |
| 5 | **N** Nonce and RNG | R-38, R-39↓ |
| 5 | **I** DoS, gaming, relay | R-40, R-41, R-42 |
| 6 | **J** Economics, carbon | R-43, R-44, R-45, R-46, R-63? |
| 6 | **T** Crypto agility | R-61, R-62 |
| — | **F** Hardware envelope | R-22 (literature half only) |

**Closed:** R-01, R-02, R-03 (→ R-47), R-04, R-05*, R-06, R-07, R-08, R-09, R-10, R-20, R-21, R-49,
R-60.*R-05 (batch verification across owners) is retained in Domain A's search notes but not
scheduled — ADR-060's row-count work removed its urgency.

**Active: 57 topics across 19 domains.** Eight are new since v2 and exist to retire assumptions
rather than to close gaps — which is the point of this revision.
