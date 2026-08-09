# Design Interrogation — ADR-001 to ADR-015

**Scope:** unbiased re-read of the first fifteen ADRs against their own cited evidence and their own arithmetic.
**Method:** for each decision — (1) why was it taken, (2) was the research sufficient, (3) why were the alternatives rejected. Claims were recomputed where they were numeric.
**Output:** 27 findings, each routed to one of three destinations.

| Route | Meaning | Count |
| --- | --- | --- |
| `[COUNCIL]` | Genuine design conflict. Cannot be settled by reading — needs the design-council skill or an owner ruling. | 13 |
| `[RESEARCH]` | The decision may be right, but the evidence for it does not exist in the corpus. Feeds `reading-list-v2`. | 7 |
| `[CORRECT]` | Documentation or arithmetic error. Fix in place, no decision change. | 7 |

**Headline:** the ADR corpus is unusually disciplined about *sourcing* — nearly every claim carries a paper reference — and unusually weak about *checking*. Several load-bearing numbers do not reproduce from their own stated inputs, and the single most important primitive in the system (the storage proof) is specified in two mutually incompatible ways across ADR-002 and ADR-014.

---

## Band A — Blockers

These change what gets built. Nothing downstream of them should be treated as settled.

### F-01 `[COUNCIL]` The verifier cannot verify. ADR-002 and ADR-014 specify two different, incompatible proofs.

ADR-002 is titled *"PoR Merkle Challenge"* and its references discuss Merkle path verification. But ADR-014 Defence 4 states the response is `SHA256(chunk_data || challenge_nonce)`, *"verified by the microservice which independently has the expected hash."*

These cannot both be true. Three properties are asserted simultaneously:

1. The nonce is unbounded and generated fresh at challenge time (ADR-002 — this is the whole point of rejecting Storj v2's pre-generated scheme).
2. The response is a hash over the full chunk body.
3. The microservice does not store chunk bodies.

Any two are satisfiable; all three are not. To check `SHA256(chunk_data || nonce)` for an arbitrary fresh nonce, the verifier needs `chunk_data`. A stored digest of the chunk does not help — hashing is not composable that way.

The only reconciliation consistent with ADR-002's title is a real Merkle scheme: build a tree over the chunk's sub-blocks at upload, store only the root, derive the challenged leaf indices deterministically from the nonce, and have the provider return leaves plus authentication paths. That is a materially different wire format, a different `audit_receipts` schema, and a different response-size and deadline calculation from what ADR-014 and ADR-017 currently describe.

**Question for the council:** which mechanism is actually intended, and is anyone aware the two ADRs disagree?

**Compounding issue:** the entire proof-of-storage primitive literature is absent from Papers 01–58. There is no Ateniese (PDP), no Juels–Kaliski (PoR), no Shacham–Waters (compact PoR). The system read the *deployments* (Storj, Filecoin, Tahoe-LAFS) and skipped the *primitives* those deployments implement. This is the largest single gap in the corpus and it sits directly under the feature the product's trust story depends on. → Reading list Domain A.

---

### F-02 `[COUNCIL]` The audit scalability ceiling is hit inside the V2 launch envelope, not at V3.

ADR-002 states full daily audit remains feasible *"to approximately 100,000 providers × 10,000 chunks"* against a Postgres ceiling of 5,000–10,000 INSERT/sec.

Two problems.

First, the stated scale already exceeds the stated ceiling: 100,000 × 10,000 = 10⁹ rows/day = **11,574 rows/sec**.

Second, and worse, 10,000 chunks × 256 KB = **2.5 GB per provider**. That is not a desktop provider sharing idle disk; it is a rounding error. Against the ~70 GB conservative tier the implementation actually assigns:

| Providers | Per-provider | Chunk-audits/day | Sustained INSERT/sec |
| --- | --- | --- | --- |
| 500 | 70 GB | 143 M | 1,659 |
| 1,000 | 70 GB | 287 M | 3,319 |
| 1,000 | 200 GB | 819 M | **9,481** |
| 5,000 | 70 GB | 1.43 B | **16,593** |

The ceiling is reached at roughly **1,000 providers holding 200 GB each** — a plausible year-one target, not a V3 problem. Full daily audit of every chunk is not a viable steady state.

The fix is probabilistic spot-checking with a formal detection bound, which is exactly what the missing PoR literature provides (Juels–Kaliski give explicit `(ρ, δ)` guarantees for sentinel sampling). But see F-03.

---

### F-03 `[COUNCIL]` Payment basis and audit scalability are coupled, and nothing acknowledges it.

ADR-012 pays providers **per audit passed**. F-02's fix is to stop auditing every chunk. Those are the same lever.

Under sampling, a provider's income becomes a function of how often they were sampled — which is a microservice-controlled random variable, not a measure of service delivered. Two providers storing identical data earn different amounts that month. That is a direct fairness and trust problem for a product whose provider-facing pitch is predictable income (ADR-045 exists specifically to make earnings legible).

There are clean answers — pay per `chunk-GB-period` and use audits purely as a *gate* rather than a *meter*; or pay per sampled audit but normalise by sampling rate. Neither is chosen. The council needs to settle the payment unit before the audit scheduler is finalised, because the schema follows from it.

Also unresolved in ADR-012 as written: a provider storing 1 GB as 4,000 chunks earns 4,000 audit-units; the same gigabyte in fewer, larger chunks earns fewer. There is no stated normalisation by chunk size.

---

### F-04 `[CORRECT]` + `[COUNCIL]` The background bandwidth budget is stated in two units that differ by 8×.

- ADR-009 (the ADR that *defines* the budget): **"The 100 KB/s background upload bandwidth assumption."**
- ADR-003: "BWavg ≈ 39 Kbps/peer — well within 100 KB/s background budget."
- ADR-004: "At 100 Kbps per peer, repair completes in ≈ 8 h."
- ADR-010: "≈ 130 Kbps/peer, exceeding the 100 Kbps background budget (ADR-009)."

100 KB/s = 800 Kbps. Every consumer of this budget except ADR-003 reads it as 100 Kbps. This is not cosmetic — see F-05 and F-08.

---

### F-05 `[COUNCIL]` The quantitative case for excluding mobile providers dissolves under F-04.

ADR-010's stated conclusion is unambiguous: *"Mobile providers are excluded on bandwidth grounds alone"* — because BWavg at 90-day MTTF ≈ 130 Kbps/peer exceeds "the 100 Kbps background budget."

If the budget is what ADR-009 says it is — 100 KB/s, i.e. 800 Kbps — then 130 Kbps is **16% of budget** and the bandwidth argument does not merely weaken, it inverts.

The decision to defer mobile may well survive on other grounds; ADR-009's point about OS background-execution limits is real and independent. But the headline argument, the one presented as decisive, is currently void. An ADR whose stated reasoning fails should not remain in `Accepted` with that reasoning intact.

---

### F-06 `[COUNCIL]` ADR-010 conflates availability with MTTF throughout.

The mobile evidence cites Paper 20's finding that 87.6% of unincentivised sessions last under 8 hours, and Paper 08's 6.4 joins/leaves per host per day, as evidence of *"MTTF ~1 month."*

Session length and mean time to permanent failure are different random variables. A phone online 8 h/day for two years has an MTTF of two years and an availability of 33%. Vyomanaut's architecture already separates these cleanly — lazy repair plus the 72-hour departure threshold (ADR-004, ADR-006) exists precisely so that diurnal absence does **not** count as failure. Feeding session-length data into an MTTF slot double-penalises mobile for the exact behaviour the repair protocol was designed to absorb.

The right input is *permanent abandonment rate under financial incentive*, which nobody has measured for any device class. → Reading list Domain D.

---

### F-07 `[CORRECT]` RS(16,56) storage overhead is misstated, and the comparison to replication is backwards.

ADR-003 claims **"2.5× storage overhead vs 3× for simple replication."**

| Scheme | Total stored / data | Parity / data |
| --- | --- | --- |
| RS(16,56) | **3.5×** | 2.5× |
| 3× replication | 3.0× | 2.0× |

Under either convention applied consistently, **RS(16,56) costs more storage than 3× replication.** The ADR compares RS's parity ratio (2.5) against replication's total ratio (3.0) and reports a win that does not exist.

This matters beyond tidiness: storage expansion is the input to provider cost-per-GB and owner price-per-GB. If the unit economics were built on 2.5×, they are 40% off.

The honest justification for r=40 is durability and repair bandwidth — Giroire's `∂BWavg/∂r = 0` optimum — not storage efficiency. RS(16,56) buys a far better loss rate and much cheaper repair than 3× replication, at slightly *higher* storage cost. Say that.

---

### F-08 `[COUNCIL]` The repair-window safety margin does not reproduce.

ADR-004: Qpeek ≈ 793 GB total for N=1000; *"repair completes in ≈ 8 h — within the 12-hour reconstruction window θ. The 12-hour window provides a 4-hour margin."*

| Assumed per-peer rate | Aggregate | Time for 793 GB |
| --- | --- | --- |
| 100 Kbps | 12.5 MB/s | **18.9 h** — exceeds θ |
| 100 KB/s | 100 MB/s | **2.4 h** |

Neither reading yields 8 hours. Under the ADR-004 unit reading, reconstruction *overruns* the safety window by 57%, and the claimed 4-hour margin is a 7-hour deficit. Under the ADR-009 unit reading there is a 5× margin. The correct answer is probably the second, but "probably" is not a durability argument. Recompute and restate.

---

## Band B — Architectural conflicts

### F-09 `[COUNCIL]` ADR-001 and ADR-005 disagree on what Kademlia is for.

ADR-001 assigns the DHT: *"provider discovery, chunk location (FIND_VALUE), and replication candidate selection (FIND_NODE)."*

ADR-005 states the opposite: *"Vyomanaut retains Kademlia for chunk-address lookup only (not provider discovery, which goes through the microservice). This distinction must be preserved — Kademlia is not a provider directory."*

Provider discovery and FIND_NODE-based candidate selection are in the DHT's scope in one Accepted ADR and explicitly out of scope in another. This needs a ruling, and the losing ADR needs an addendum.

---

### F-10 `[COUNCIL]` + `[RESEARCH]` Nobody has established that the DHT earns its place.

ADR-013 makes the microservice the single authoritative coordinator for chunk placement. The microservice therefore already holds the complete, strongly-consistent chunk→provider mapping — it must, to dispatch audits and drive repair.

Given that, the DHT is a second, weaker, eventually-consistent copy of a table that exists anyway. Against a single indexed Postgres lookup it costs: k-bucket maintenance, 12-hour republication traffic, disjoint-path lookups, a Sybil surface the registration gate then has to close, the HMAC key scheme (F-11), and the ADR-048 gossip layer built to keep it coherent.

The stated benefit is *"self-healing peer discovery; logarithmic lookup in O(log n) hops."* At N=1000 that is roughly 10 network round-trips versus one indexed query. O(log n) is only a win once n is large enough that the constant factors stop dominating, and that crossover has never been computed for this system.

This is not an argument that the DHT is wrong. It is an argument that **the most complex subsystem in the architecture is justified by an asymptotic argument nobody has instantiated with real numbers.** → Reading list Domain H.

---

### F-11 `[COUNCIL]` The DHT privacy scheme and the repair path are in direct conflict.

ADR-001 makes `HMAC(chunk_hash, file_owner_key)` the DHT lookup key and calls it *"an active design requirement"* that closes a field-wide privacy gap: *"Only the file owner can reverse-map a DHT key to its chunk."*

But the availability service republishes every key-value pair on a 12-hour cycle, and repair is triggered by the microservice — typically while the owner is offline. Both require computing the key. So either:

- **the microservice holds owner keys**, in which case the privacy property is only against third parties, not against Vyomanaut, and the ADR must stop claiming otherwise; or
- **it does not**, in which case republication and microservice-driven repair cannot use the DHT — and the DHT is confined to owner-initiated retrieval only, which sharpens F-10 considerably.

Secondary consequence: HMAC keying destroys content-addressing, so cross-owner deduplication becomes impossible by construction. That may be an acceptable or even desirable privacy trade, but it should be a stated consequence rather than a silent one.

---

### F-12 `[CORRECT]` + `[RESEARCH]` "Quorum reads prevent DoS" is not a mechanism.

ADR-001: *"To prevent DoS on the central microservice: quorum read mechanism for consistency (latency trade-off accepted)."*

Quorum reads give consistency across replicas. They do not mitigate denial of service — under a volumetric or application-layer flood, requiring 2 of 3 replicas to answer makes the service *more* fragile, not less, because it raises the per-request cost. DoS resistance comes from rate limiting, admission control, client puzzles, anycast absorption, and cost asymmetry.

ADR-014 repeats the claim: SoK Challenge 2 (DoS on the coordination entity) is described as *"mitigated by the (3,2,2) quorum."* It is not.

Nothing in Papers 01–58 covers DoS resistance for a coordination service. For an architecture whose own risk register names the microservice as its central dependency, that is a conspicuous hole. → Reading list Domain I.

---

### F-13 `[COUNCIL]` The S/Kademlia parameters are calibrated for a threat model the registration gate removes.

ADR-001 replaces S/Kademlia's cryptographic node-ID puzzles with *"registration acts as certificate authority"* — then imports S/Kademlia's parameters wholesale, including *"d=4,8 — maintains 99% efficiency at 30% adversarial node share."*

That 30%-adversarial figure is derived for a network with **no admission control**. If KYC gating works, 30% adversarial share is not the operating point and d=4 disjoint paths are over-engineering. If it does not work, then account-UUID-derived node IDs are strictly weaker than the puzzles they replaced, and d=4 is under-engineering. The defence is being counted twice.

Separately: `d=4,8`, `k=8,16` — an Accepted ADR should not carry unresolved ranges for parameters that must be compiled into the routing table.

---

### F-14 `[CORRECT]` The vetting-period derivation produces 2.6 months, not 4–6.

ADR-005: *"a storage node's estimated audit success probability exceeds 99% after 80 consecutive successful audits… The 4–6 month vetting period is calibrated to accumulate at least 80 clean audit events at the 24-hour polling interval."*

80 audits at one per day is 80 days ≈ **2.6 months**. Either the polling rate assumed here is not 24 h, or 4–6 months is justified by something other than the 80-audit threshold. As written the stated derivation contradicts the stated conclusion.

---

### F-15 `[COUNCIL]` + `[RESEARCH]` The vetting statistic measures the wrong variable.

Even at the right duration: 80 consecutive audit passes, via a Jeffreys prior on a Bernoulli, gives confidence about a provider's **audit success rate**. It says essentially nothing about their **retention** — whether they will still be online in 180 days.

But retention is what every downstream parameter depends on. MTTF ∈ [180, 380] days is the input to ADR-003's r=40, ADR-004's r0=8, and ADR-010's entire tier-collapse argument. Vetting is being used to certify MTTF using a statistic that does not estimate MTTF.

Estimating lifetime from a truncated observation window is a solved problem in the right field — survival analysis with right-censored data. None of it is in the corpus. → Reading list Domain D.

---

### F-16 `[COUNCIL]` + `[RESEARCH]` The churn model is 26-year-old data from the wrong country and the wrong hardware class.

ADR-006 and ADR-007 derive the 72-hour departure threshold from Bolosky's bimodal distribution: nightly µ=14 h σ=1.9 h, weekend µ=64 h σ=2.1 h.

Bolosky et al. measured **corporate desktops on the Microsoft campus LAN in 1999**. Vyomanaut targets Indian consumer desktops in 2026 — different diurnal patterns, different power reliability, laptops with lids, shared family machines, and grid interruptions in large parts of the addressable market. σ=1.9 hours on a nightly absence is implausibly tight for consumer hardware even in 2000.

This is the clearest instance of the corpus's structural problem: it is not that the papers are old, it is that **measurement papers expire and primitive papers do not**. Kademlia (2002), Dynamo (2007) and AONT-RS (2011) age fine — they describe mechanisms. Bolosky (2000), Saroiu (2002) and Blake & Rodrigues (2003) describe a *population*, and that population no longer exists. Three separate 20-plus-year-old measurement papers are currently load-bearing for V2 launch parameters. → Reading list Domain D.

---

### F-17 `[CORRECT]` The RTO formula is dimensionally inconsistent.

ADR-006: *"RTO = AVG + 4×VAR, where AVG and VAR are the EWMA mean and variance."*

TCP's actual formula (RFC 6298) is `RTO = SRTT + 4·RTTVAR`, where `RTTVAR` is a smoothed **mean deviation** — same units as the mean. Variance is in squared units; adding it to a mean is not meaningful and, at any realistic jitter, produces a wildly inflated timeout. The implementation (`computeRTO`) should be checked against whichever the spec actually means.

---

### F-18 `[COUNCIL]` Defence 2 fails exactly in the region Defence 1 permits.

ADR-014's outsourcing deadline is `(chunk_size / p95_throughput) × 1.5` — a **network throughput** bound. It defeats an accomplice across a WAN with meaningful RTT.

It does not defeat an accomplice on the same LAN or in the same datacentre, where fetch latency is sub-millisecond and comfortably inside any deadline derived from a consumer uplink.

And Defence 1 explicitly permits up to **20% of a file's shards (≈11 of 56) within a single ASN**. So the attack Defence 2 targets is unblocked precisely in the topology Defence 1 tolerates. Eleven co-located "providers" backed by one disk is a cheap, in-scope attack against which neither defence bites. That is a structural gap between two defences that are each individually reasonable.

---

### F-19 `[CORRECT]` `p95` is defined as two opposite tails in the same paragraph.

ADR-014 defines `p95_measured_upload_throughput` as *"the 95th percentile of upload throughput"* and then, two lines later, as *"the 95th-percentile slowest observed response."* The 95th percentile of throughput is a **fast** value; the 95th percentile of latency is a **slow** one. They produce deadlines differing by a large factor and in opposite directions. The worked example (500 KB/s → 768 ms) implies the fast reading; the prose implies the slow one.

---

### F-20 `[COUNCIL]` The JIT anomaly detector systematically penalises the best hardware.

ADR-014 flags a response as just-in-time retrieval if `response_latency_ms < (chunk_size / p95_throughput) × 0.3`.

New providers are initialised to the **pool median**. A provider on NVMe with a good uplink will legitimately respond several times faster than the pool median — and will be flagged. Three flags in seven days applies a 0.5× weight to their audit passes for thirty days, i.e. halves their income.

So the detector's most likely victims are exactly the high-quality providers the Preference subsystem is trying to attract. The 0.3 threshold carries no derivation and no false-positive analysis. At minimum it needs to be relative to that provider's own observed distribution rather than the pool's, and it needs an FPR estimate before it gates payment.

---

### F-21 `[CORRECT]` ADR-014 contains stale and miscounted content.

- The title says *"Four Adversarial Provider Defence Classes"*; the body defines five.
- Defence 1 still carries *"Activate only after 5 × n shards exist in the network."* ADR-005 records that this clause was **retired** and superseded by ADR-029's readiness gate (≥56 vetted providers across ≥5 ASNs). The correction was applied in ADR-005 and never propagated here. An implementer reading ADR-014 alone will build the retired rule.

---

### F-22 `[COUNCIL]` + `[RESEARCH]` V2's audit trail provides less verifiability than the blockchain it replaces, and the replacement is deferred.

ADR-015's provider receipt is *"stored locally in a receipt store on the provider's machine."* A log held solely by the party it protects is not evidence — the provider can delete inconvenient entries, and cannot prove that a countersignature was *withheld*. The microservice can decline to countersign and the provider has no recourse. The ADR concedes this: *"Providers must trust the microservice in V2."*

The product's framing is that a non-crypto mechanism reproduces blockchain's three functions. Function 3 (public dispute resolution) is deferred to V3 — so at launch the system offers strictly weaker verifiability than the thing it replaced, while marketing parity.

The V3 design has its own gap. Certificate Transparency is correctly cited as the model, but CT's actual security rests on **gossip between verifiers** to detect split views. A daily published Merkle root with no gossip protocol lets an operator serve different roots to different providers and remain undetected indefinitely. The transparency-log gossip literature is well-developed and entirely absent from the corpus. → Reading list Domain G.

---

### F-23 `[CORRECT]` The append-only guarantee has a carve-out that defeats it.

ADR-015 claims the audit table is *"INSERT only, no UPDATE/DELETE, enforced at DB level"* — then adds *"audit_result column specifically is treated as an exception allowing UPDATE."*

Any UPDATE path is a tamper path, and role-level SQL grants cannot distinguish a legitimate PENDING→PASS transition from a malicious PASS→FAIL rewrite. The standard fix is to make tampering **detectable** rather than prevented: hash-chain each row to its predecessor, or write completions as a second append-only row rather than mutating the first. Neither is specified.

---

### F-24 `[COUNCIL]` A four-day power cut costs a provider six months of standing and their escrow.

ADR-007: >72 h silent absence → escrow **seized**, status DEPARTED, assignments removed. ADR-007 further specifies that rejoining creates a new `provider_id` with **no score history carried over** — meaning the full vetting period restarts from zero.

For the target market this is a live false-positive, not a hypothetical. Extended grid outages, a household move, or a two-week holiday with the machine unplugged all trip it. The penalty — forfeited earnings plus a multi-month re-vetting climb — is severe, irreversible, and indistinguishable from the penalty for genuine abandonment.

It also creates a perverse incentive: because *announced* departure releases escrow while *silent* departure seizes it, the rational move for a provider facing uncertain downtime is to announce a permanent exit and re-register later. That is the opposite of the retention behaviour ADR-054 is designed to produce.

Worth asking separately whether punitive forfeiture of earned funds is enforceable at all under Indian consumer and payment-aggregator rules — ADR-011 already notes Vyomanaut does not qualify for Razorpay Escrow+ and routes around it via Route holds. A payment-API hold is a mechanism, not a legal right to forfeit.

---

### F-25 `[COUNCIL]` The zero-fee argument applies to money coming in, not money going out.

ADR-011 rests on *"UPI has zero per-transaction merchant fee"* and concludes *"zero per-transaction fee makes micro-payments economically viable."*

Zero MDR applies to **collections** (data owners paying in). **Payouts** — Razorpay Route transfers and RazorpayX payouts, which is how providers get paid — carry per-transaction fees. And the payout side is precisely where ADR-012's per-audit-passed model generates high transaction volume.

The month-end `on_hold_until` aggregation in ADR-011 probably resolves this in practice, but it is mentioned as an escrow-timing detail, not as the mechanism that makes the economics work. The dependency should be explicit, because it constrains payout frequency — and payout frequency is a headline provider-experience parameter.

Also worth verifying: ADR-009 asserts *"100 Mbps symmetrical at ₹600/month"* as the Indian consumer baseline. Consumer FTTH at that price point is typically asymmetric, and **upload** is the binding constraint for a storage provider. If real upload is 20–40 Mbps, every bandwidth budget downstream needs revisiting.

---

## Band C — Evidence and hygiene

### F-26 `[CORRECT]` ADR-013's I-confluence table has three defects.

1. **`ASSIGN chunk to provider` is classified NO for "unique placement."** Placement uniqueness is not actually an invariant — ADR-007 explicitly states two providers holding the same shard is fine. The genuine non-I-confluent invariant is the **20% per-ASN threshold**, which is a bounded counter (exactly Bailis's canonical non-I-confluent case). Right answer, wrong reason.
2. **"6 coordinated operations create a bottleneck dependency on the payment microservice."** Score-floor enforcement, chunk placement, and token validation are not payment operations.
3. **"Out of 20 core operations"** is presented as exhaustive but omits at least: repair job scheduling, capability token *issuance* (as distinct from validation), DHT republication, vetting graduation, escrow hold-release scheduling, and `jit_flag` aggregation. Call it a representative sample, or complete it.

### F-27 `[RESEARCH]` The 5% CPU budget rests on one paper and no measurement of the actual workload.

ADR-009 has the thinnest evidence base of the fifteen — a single source (Bolosky 2000), which measured *ambient desktop load*, not Vyomanaut's workload.

The real per-provider cost is RS(16,56) encode/decode over GF(2⁸) with 40 parity shards, SHA-256 across 256 KB chunk bodies per audit, ChaCha20-Poly1305 on the transfer path, plus a Wails/WebView2 GUI process. None of this has been estimated, let alone measured on a min-spec Indian desktop. The 5% figure is asserted.

Encoding throughput is well-studied and directly measurable from the literature (ISA-L, Jerasure, Cerasure and the newer bit-matrix work all publish comparable numbers). → Reading list Domain F.

### F-28 `[COUNCIL]` + `[RESEARCH]` The bootstrap network is the worst case for the durability model, and it was never modelled.

RS(16,56) places 56 shards per file. ADR-029 gates first upload at **≥56 vetted providers across ≥5 ASNs**.

At that exact readiness point, every file lands on essentially every provider in the network. Failure independence — the assumption under Giroire's LossRate — is at its weakest precisely when the network first accepts data. And the 20% ASN cap with 5 ASNs allows 11.2 shards per ASN; 5 × 11.2 = 56, so the cap is **exactly saturated** at bootstrap and constrains nothing.

ADR-003's correlated-failure argument (via Paper 38) computes the worst case as 11 shards lost, leaving 44 — 28 above the s=16 floor. That holds at steady state with many ASNs. It does not hold at N≈56 across 5 ASNs, where a single ASN outage removes the maximum the cap permits and the remaining ASNs are carrying the rest of every file simultaneously.

Nobody has run the durability numbers for the regime N ∈ [56, ~500]. That regime is where the product spends its first year. → Reading list Domain E.

### F-29 `[RESEARCH]` n=56 is wide-stripe territory and no wide-stripe literature is in the corpus.

ADR-003's own reference note concedes it: *"n=56 is wide-stripe territory (3× larger than any surveyed production system)."* That observation is recorded and then not acted on.

Wide stripes have a specific, well-documented failure mode — repair fan-in scales with k, so a single shard loss touches 16 providers, and full-node recovery touches vastly more. There is an active research line on exactly this (combined locality, wide-stripe repair, I/O-efficient repair constructions). None of it has been read. → Reading list Domain C.

### F-30 `[RESEARCH]` The small-file penalty is unquantified because the file-size distribution was never stated.

s=16 × 256 KB means the minimum efficiently-coded object is **4 MB**. Below that, padding dominates: a 200 KB file becomes 56 shards of 256 KB = 14 MB stored, a 70× expansion.

No ADR states an assumed file-size distribution, and no ADR specifies small-file handling (bundling, sub-striping, or a replication fallback below a size threshold). For a consumer product, small files are the common case. → Reading list Domain C.

### F-31 `[COUNCIL]` + `[RESEARCH]` ADR-008 defers the parameter that determines provider income, and never analyses gaming.

*"Exact weights are a product decision and should be tuned empirically after launch."*

The weights combine the 24 h / 7 d / 30 d windows into the score that drives assignment (ADR-005 Preference) and therefore income. Deferring them defers the incentive design itself.

More concerning: there is no adversarial analysis of the scoring function. If the 24-hour window carries the highest weight, a provider can optimise for burst availability around audit windows and be scored well while providing poor sustained service. On-off and oscillation attacks against reputation systems are a mature literature; the corpus has EigenTrust (2003) and PeerTrust (2004) and nothing since. → Reading list Domain I.

---

## What holds up

Worth stating plainly, because the above is unrelenting:

- **ADR-013 (I-confluence)** is the strongest ADR in the set. Applying a formal test to every operation, and writing down the procedure for testing future ones, is genuinely good practice and rare at this stage.
- **ADR-015's idempotent retry protocol** — PENDING row before validation, unique index on nonce, NULL-tolerant scoring, 48 h GC — is careful, correct, and eliminates a Kafka dependency for the right reason. The critique in F-22/F-23 is about the trust model surrounding it, not this mechanism.
- **ADR-012's rejection of ingress payment** on the delete-and-restore attack is exactly right and well-sourced.
- **ADR-011's rejection of crypto payment** is correct, commercially and technically, and the `PaymentProvider` abstraction is the right hedge.
- **ADR-004's lazy repair** is well-reasoned and the priority ordering from Paper 39 (permanent departures ahead of transient) is a real insight, not a citation ornament.
- The **cross-referencing discipline** — ADR-003 declaring ADR-014 a hard co-requisite for its own durability claim — is unusually mature. Most projects discover that dependency in production.

---

## Routing summary

**To the design council, in dependency order:**

1. **F-01** — settle the storage proof. Everything in the audit, receipt, payment and scoring path is downstream.
2. **F-02 / F-03** — audit scale and payment unit, together. They cannot be settled separately.
3. **F-04 / F-05 / F-08** — fix the bandwidth unit, then re-rule on ADR-010 and re-derive the repair window.
4. **F-09 / F-10 / F-11** — DHT scope, necessity, and privacy-vs-repair. One session; they are the same question from three angles.
5. **F-18 / F-20** — outsourcing defence and the JIT detector.
6. **F-22** — the V2 trust story, before it is used in provider-facing copy.
7. **F-24 / F-25** — provider penalty policy and payout economics.
8. **F-28 / F-31** — bootstrap durability regime; scoring weights and gaming.

**Documentation corrections (no ruling needed):** F-07, F-14, F-17, F-19, F-21, F-23, F-26.

**To the reading list:** F-01, F-06, F-12, F-15, F-16, F-27, F-29, F-30 — mapped to domains in `reading-list-v2.md`.
