# Design Interrogation — ADR-016 to ADR-030

**Continues** [`adr-001-015-design-interrogation.md`](./adr-001-015-design-interrogation.md). Finding numbers continue from F-31.

**Method unchanged:** why was the decision taken, was the research sufficient, why were alternatives rejected. Numeric claims recomputed.

| Route | Count |
| --- | --- |
| `[COUNCIL]` | 10 |
| `[RESEARCH]` | 5 |
| `[CORRECT]` | 6 |

**Headline:** this block is, on average, *better* work than ADR-001–015. ADR-027, ADR-028, ADR-029 and ADR-030 each identify a real defect in an earlier ADR and fix it properly — that is a healthy corpus correcting itself. ADR-023 is the strongest technical ADR in the project.

But two things go the other way. First, **ADR-030 states in plain text that the audit cannot verify anything** — turning F-01 from an inferred contradiction into a documented one. Second, ADR-022 leans on the 20% ASN cap for *confidentiality* without noticing that the confidentiality margin is 5 shards where the durability margin is 29.

---

## Band A — Blockers

### F-32 `[COUNCIL]` ADR-030 confirms the audit verifies nothing. F-01 is not a doc conflict — it is a live defect.

ADR-030 §"Synthetic chunk generation" states it outright:

> *"The microservice cannot verify the response hash directly (same as real chunks — it never holds the chunk data). The Ed25519 signature verification and the economic deterrence of score penalties together remain the validation mechanism."*

Compare ADR-014 Defence 4, which is still `Accepted` and unamended:

> *"Without the chunk, the provider cannot compute a valid response. Verified by the microservice which independently has the expected hash."*

And ADR-017's field-rationale table, which lists `response_hash` as the mitigation for exactly that attack.

ADR-030's parenthetical **"(same as real chunks)"** is the load-bearing phrase. It confirms the gap is systemic, not specific to vetting.

**What this means operationally.** A provider deletes every chunk, keeps their Ed25519 key, and answers every challenge with 32 random bytes. The signature verifies — it proves *who sent it*, never *what they hold*. The microservice has no expected value to compare against. The receipt is written `PASS`. The provider is paid (ADR-012, ADR-024 §1), scored (ADR-008), never repaired against (ADR-004), and never seized (ADR-024 §5).

Every mechanism downstream of the audit is currently gated on a check that cannot fail. That includes ADR-024 §7's invocation of SHELBY Theorem 1 — Condition (i) is *trivially satisfied* at `pau ≈ 1` only if the audit is a real test. At `pau ≈ 1` with a no-op audit, the equilibrium argument is vacuous.

The "economic deterrence" fallback does not close it. Deterrence requires a non-zero probability of detection, and the detection probability here is zero. Score penalties penalise providers who *fail* to answer, not providers who answer falsely — so the only behaviour actually deterred is going offline.

This is the single most important item in the project. Nothing else in either interrogation document matters as much.

**The fix is known** — a Merkle tree built over each chunk at upload, root stored by the microservice, nonce selecting leaf indices, provider returning leaves plus authentication paths. That is what ADR-002's own title says it does. Domain A of the reading list exists to specify it properly.

---

### F-33 `[COUNCIL]` Vetting audits *are* fixable, cheaply, and should be fixed regardless of how F-32 resolves.

For synthetic chunks the microservice is the data's author, so the impossibility in F-32 does not apply. ADR-030 chooses to discard the data — but if synthetic chunk content were derived rather than random:

```
syntheticData = ChaCha20(key = HKDF(vetting_seed, provider_id || chunk_index), nonce = 0)
```

the microservice could regenerate any chunk on demand and verify `SHA-256(syntheticData ‖ nonce)` exactly. Cost: 32 bytes of seed instead of nothing, plus one keystream generation per verification.

Two consequences worth noting. It makes vetting a genuine honesty test — which is what vetting is *for*. And it produces the odd situation where vetting audits are cryptographically sound while post-graduation audits are not, which is a useful forcing function for F-32.

Also fixes the 146 GB launch-time transfer ADR-030 concedes: derived data can be regenerated provider-side from a per-chunk seed rather than transmitted.

---

### F-34 `[COUNCIL]` Two colluding ASNs recover plaintext. The confidentiality margin is 5 shards; the durability margin is 29.

ADR-022 asserts:

> *"The 20% ASN cap (ADR-014) ensures that no single correlated provider group can hold more than ~11 of 56 slices, keeping the threshold safe."*

That is true for **one** group. AONT-RS's all-or-nothing property collapses the moment an adversary assembles `k=16` slices — at which point they recover `h`, then `K`, then the full plaintext.

| Colluding ASNs | Shards held | Plaintext recoverable |
| --- | --- | --- |
| 1 | 11 | No |
| **2** | **22** | **Yes** |
| 3 | 33 | Yes |

The asymmetry is the point. ADR-003 and ADR-014 justify the 20% cap on durability grounds, where the numbers are comfortable — lose one ASN entirely and 45 shards survive against a floor of 16, a slack of 29. Confidentiality runs the opposite direction: you need only 16 shards, and one ASN already supplies 11. **Slack of 5.**

At the ADR-029 bootstrap gate this is acute. Five ASNs minimum means two of them is 40% of the network's entire diversity — a single large residential ISP plus one cloud provider, or two providers who registered from the same corporate network under different ASNs. The cap was never designed as a confidentiality control and does not function as one.

Structurally the two thresholds are independent and should be decoupled: the *erasure* threshold `k=16` is set by durability and repair economics; the *confidentiality* threshold should be set by placement diversity. Splitting `K` under a separate higher-threshold secret-sharing scheme, or raising the effective disclosure threshold above the reconstruction threshold, are both standard moves. Neither is considered.

Note this interacts with F-28: at bootstrap the cap is exactly saturated, so 5 ASNs × 11 shards ≈ 56 and *any* two of the five suffice.

---

### F-35 `[COUNCIL]` There is a single-writer service inside a leaderless cluster, and no mechanism to elect or fail it over.

ADR-025 adopts Dynamo's model: N=3, R=2, W=2, gossip membership, local failure detection, explicitly **no external consensus** ("no etcd/Consul to operate").

It then states that the six non-I-confluent operations from ADR-013 — escrow debit, escrow seizure, score floor, chunk placement uniqueness, token validation, physical-delete prohibition — *"are handled by the single authoritative payment/assignment service, not by quorum vote."*

Which replica is that service? What happens when it dies? How is a successor chosen without split-brain?

Nothing in ADR-025 answers this. Gossip failure detection tells replicas that a peer stopped responding; it does not confer the right to take over its role, and two replicas that each conclude the third is dead can both proceed.

**Dynamo is leaderless precisely because every Dynamo operation is mergeable.** Shopping carts reconcile; sibling versions are resolved by the client. Vyomanaut imported Dynamo's membership layer while retaining six operations that are *not* mergeable — and the consequences of getting them wrong are a double escrow release or a violated 20% ASN cap, not a resurrected cart item.

Related gap: ADR-025 never specifies conflict resolution for the quorum path either. R+W>N prevents stale reads only in combination with a versioning scheme; Dynamo uses vector clocks plus explicit reconciliation. ADR-025 mentions neither.

The honest options are (a) accept an external consensus dependency for the six operations, (b) implement lease-based single-writer election with a fencing token, or (c) demonstrate that all six can be made mergeable. Any is defensible; silence is not.

---

## Band B — Architectural conflicts

### F-36 `[COUNCIL]` The AONT cipher is chosen by the *encoder's* hardware and recorded nowhere. Cross-device decode breaks.

ADR-019 specifies two AONT paths — ChaCha20-256 without AES-NI, AES-256-CTR with it — selected by CPUID *"at daemon startup… never re-check at runtime."*

The decoder must apply the same cipher the encoder used. But the pointer file schema (ADR-020) has no cipher field: it carries `schema_version`, `file_id`, `erasure_params`, `segments`, `owner_sig`. And ADR-022's decode pipeline hardcodes `d_i = c_i XOR AES-256(K, i+1)`.

So a file uploaded from a modern desktop (AES-NI → AES-CTR) and later recovered on the owner's older laptop (no AES-NI → ChaCha20) decodes to garbage. The canary check fires and ADR-022 Step 4 escalates it to the audit subsystem as *corruption* — misattributing a client-side format mismatch to provider failure.

This is squarely the "Scenario 1 — Device loss" recovery path ADR-020 is built around.

Fix is a one-byte `aont_cipher` field in the pointer file and in the AONT package header, set at encode time and honoured at decode. Worth doing before any file is written, since the pointer file is versioned but existing files can't be retro-tagged.

---

### F-37 `[CORRECT]` ADR-014 and ADR-017 invert the JIT signal.

- ADR-014: a response is flagged when `response_latency_ms < (chunk_size / p95_throughput) × 0.3` — *"flagged as anomalously **fast** (consistent with JIT retrieval from a co-located node)."*
- ADR-017 field table: *"JIT retrieval detector — anomalously **high** latency flags just-in-time retrieval."*

Opposite directions on the same column. ADR-014's reasoning is the coherent one (a co-located accomplice answers faster than local disk would predict); ADR-017 appears to have been written against an earlier model where fetching from elsewhere was assumed slower. One is a live implementation instruction and the other is a live schema document.

Related to F-19 and F-20 — this detector now has three separate specification defects.

---

### F-38 `[CORRECT]` ADR-017's schema cannot express ADR-015's protocol.

ADR-015's idempotent retry protocol — the strongest single mechanism in the first fifteen — requires:

> *"audit_result column must accept NULL (in-flight) in addition to PASS/FAIL/TIMEOUT. Add: `abandoned_at TIMESTAMPTZ NULL`. Add: UNIQUE INDEX on challenge_nonce."*

ADR-017 still declares `audit_result ENUM(PASS, FAIL, TIMEOUT) NOT NULL`, has no `abandoned_at`, and has no unique index on `challenge_nonce`.

ADR-015 correctly flagged the change as "Schema change required" and it was never propagated to the ADR that owns the schema. An implementer building from ADR-017 gets a table in which the PENDING-row protocol is impossible.

---

### F-39 `[CORRECT]` The audit deadline formula is superseded in one ADR and still live in two others.

ADR-014 replaced self-declared upload speed with measured throughput, for a good reason (Paper 21: 30% of peers misreport bandwidth) — deadline is `(chunk_size / p95_measured_upload_throughput) × 1.5`.

Still using the retired form:
- **ADR-002:** `(chunk_size / declared_upload_speed) × 1.5`
- **ADR-023:** same formula, and derives the ~614 ms figure from *"5 Mbps declared speed"* — a number that then propagates into the HDD benchmark pass criteria and the ADR-021 relay-overhead analysis.

The 614 ms constant is used as a fixed budget across three ADRs, but under ADR-014 the deadline is per-provider and varies with measured p95. The storage engine's latency budget is therefore anchored to a value the security model no longer produces.

---

### F-40 `[COUNCIL]` + `[RESEARCH]` Daily full audit costs the *provider* about an hour of disk seeking per day. It is not in the 5% budget.

ADR-023's audit path is well designed: Bloom filter, one random vLog read, `content_hash` verify, response hash. One random read per challenge.

At the ~70 GB conservative tier, that provider holds **286,720 chunks**, each audited daily:

| Device | Seek time/day | Also: SHA-256 over |
| --- | --- | --- |
| 7200 RPM HDD (~12 ms) | **0.96 h/day** | 70 GB/day |
| SSD (~1 ms) | 0.08 h/day | 70 GB/day |

ADR-023 itself notes *"most Indian home desktop providers at V2 launch will have HDDs."* An hour a day of continuous random seeking, plus hashing 70 GB, on a shared family machine is not a background process — it is audible, it competes with everything the owner does, and it wears the drive.

The ≤5% CPU budget (ADR-009) covers CPU. Nothing budgets audit **disk I/O**, and ADR-023's own HDD benchmark protocol tests *one* random read per second, roughly 300× below the real rate implied by daily full audit at this capacity.

This is the provider-side face of F-02. The database ceiling and the provider disk cost are the same problem, and probabilistic sampling fixes both at once. It also strengthens the case: sampling is not merely a scale optimisation, it is what makes the product tolerable to run.

---

### F-41 `[COUNCIL]` ADR-024 justifies seizure as cost recovery, then says there is no cost to recover.

§5, on silent departure: *"All earnings in the 30-day rolling window are seized (transferred to the repair reserve fund). The repair reserve fund is used to subsidise the cost of onboarding replacement providers."* And: *"the seized escrow from a departing provider will exceed the repair bandwidth cost."*

§7, two paragraphs later: *"the repair reserve floor is primarily a bookkeeping constraint, not a hard capital requirement, since repair bandwidth cost is borne by the receiving replacement provider, not by the microservice."*

If the operator bears no repair cost, seizure recovers nothing. It is a pure penalty transferring a departing provider's earned income to the operator's balance sheet.

That may still be defensible as a deterrent — but it must be argued as a deterrent, not as cost recovery, because the two have different fairness properties and, per F-24, different legal ones. A forfeiture clause justified by a cost that the same document says does not exist is not a position that survives contact with a consumer-protection complaint.

---

### F-42 `[COUNCIL]` Three independent caps stack on vetting income and no ADR multiplies them out.

| Constraint | Source |
| --- | --- |
| Synthetic chunks capped at **10%** of declared storage | ADR-030 §Storage cap |
| Release multiplier capped at **0.50** until vetting completes | ADR-024 §6 |
| Hold window doubled to **60 days** | ADR-024 §6 |
| Minimum **120 days** before ACTIVE transition | ADR-030 §ACTIVE transition |

Each is individually well-reasoned. Composed, a provider who signs up offering 200 GB earns on 20 GB, receives half of that, on a 60-day delay, for at least four months — roughly **5% of steady-state income for the first third of a year**, with the first payout landing around day 60.

The product's proposition to that person is income from idle disk. ADR-045 exists to make earnings legible and ADR-054 exists to soften the ramp — but neither computes this stack end-to-end, and ADR-030 (which added the 10% cap) does not reference ADR-024 §6 at all.

This is the number most likely to determine whether the provider network reaches the 56-provider bootstrap gate. It should be computed in rupees against a real storage rate before launch.

---

### F-43 `[RESEARCH]` ADR-026's sub-packetisation figure does not reproduce, and the correct value changes the argument.

ADR-026 rules out Clay codes because *"Clay's minimum sub-packetisation level is astronomically large (α ≥ 40^16) — computationally intractable."*

The standard MSR sub-packetisation bound at `d = n−1` is `α = (d−k+1)^⌈n/(d−k+1)⌉`. At `(n=56, k=16, d=55)`:

```
d − k + 1        = 40
⌈n / (d−k+1)⌉    = ⌈56/40⌉ = 2
α                = 40² = 1,600
```

Not 40¹⁶ ≈ 4.3 × 10²⁵. The exponent appears to have taken `k` where the formula takes `⌈n/(d−k+1)⌉`.

At α = 1,600 the sub-chunk is **164 bytes**. So the decision to rule out Clay very likely survives — but on the ADR's *second* argument (164-byte random reads on a desktop HDD are pathological), not on tractability. Those are different arguments with different futures: tractability never improves, whereas the I/O argument weakens if providers move to SSDs, which they will.

Worth flagging that this is the same failure mode as the NFR-044 interpolation formula — a build-spec expression that was wrong at its own anchor point. Formulas taken from papers and re-substituted need the substitution shown.

---

### F-44 `[COUNCIL]` Hitchhiker is now the committed V3 code family on the strength of one figure in one survey.

ADR-026 decides: *"Hitchhiker codes are the V3 repair-bandwidth candidate."* This is a structural commitment — it fixes chunk layout, the pointer file schema, and audit challenge design for V3.

The entire evidential basis is Paper 19 (EC Survey), Figure 8, reporting a 25–45% band. The Hitchhiker paper itself (Rashmi et al., SIGCOMM 2014) has never been read. Nor has any piggybacking-framework paper. The corpus contains Dimakis (Paper 55) for the bound and Goparaju (Paper 22) for MSR sub-packetisation, but nothing primary on the family actually chosen.

Given F-43 shows the *rejection* rested on a misapplied formula, the *selection* deserves at least one primary source before V3 layout work starts.

---

### F-45 `[CORRECT]` ADR-028's DHT fallback is circular.

ADR-028 §2 makes the heartbeat address primary and DHT lookup the fallback *"used only when no heartbeat has been received within 8 hours."*

But the DHT record is published **by the microservice**: ADR-021 sets *"Republication: driven by availability microservice; libp2p internal republication disabled,"* and ADR-001 assigns key-value refresh to the availability service.

So the microservice republishes into the DHT the same address it learned from the heartbeat. When the heartbeat is stale, the DHT record is stale by construction — it is a copy of the thing that went stale, at a coarser refresh interval (12 h vs 4 h).

The fallback provides no information the primary path lacks. Either providers must self-publish their multiaddr to the DHT (reversing ADR-021's disabled-republication decision), or the fallback should be dropped and the escalation table simplified.

---

### F-46 `[RESEARCH]` Server-held pointer ciphertext converts a user passphrase into an offline attack target.

ADR-020 has the microservice store `pointer_ciphertext` for every file, and calls it *"a blind store… zero-knowledge preserved even if the microservice is compromised."*

Blind, yes. But it is also an oracle. Anyone who obtains that table — an operator insider, a database breach, a subpoena — can mount an **offline** brute-force against the owner's passphrase, checking each guess by deriving `master_secret` via Argon2id, deriving `pointer_file_enc_key` via HKDF, and testing the Poly1305 tag. A correct guess is unambiguous.

Argon2id at `t=3, m=64 MB, p=4` is a reasonable interactive setting but a thin barrier against an offline attacker with GPUs, against the passphrase distribution real consumers actually choose. And the consequence of a successful guess is total: master secret → all files, forever, non-revocable.

Nothing in the corpus covers password-based KDF parameter selection under offline attack, real-world passphrase entropy distributions, or the alternatives (PAKE-derived keys so no offline-verifiable blob is ever server-held; or simply not storing pointer ciphertext server-side and pushing backup to the owner).

Note the interaction with the BIP-39 path: an owner who generates the mnemonic has a genuine 256-bit credential, but the passphrase path remains open in parallel and the system's strength is the weaker of the two.

---

### F-47 `[COUNCIL]` Two fragile nonce invariants, one well-known primitive that removes both.

**ADR-019, AONT:** ChaCha20 with `nonce = all-zeros`, safe *"because the AONT key K is chosen fresh by SecureRandom(256 bits) per segment."* Correct as stated — and it makes the entire confidentiality of every segment contingent on RNG quality at that instant. VM cloning, container image reuse, early-boot entropy starvation on low-end hardware, and snapshot/restore all produce repeated `SecureRandom` output, and the failure is silent. A random nonce costs nothing and would make K-reuse survivable.

**ADR-020, pointer file:** 96-bit counter nonce in local key store, with the daemon *"refusing to encrypt with a nonce it has used before."* It cannot enforce that — if the counter is lost or restored from a stale backup, the daemon's record of what it has used is exactly what was lost. ADR-020 concedes this as Q17-2 and offers a manual procedure.

**XChaCha20-Poly1305** removes both problems: a 192-bit nonce is safely random, so no counter state is needed and no local durability requirement exists. It is widely deployed (libsodium, age, WireGuard-adjacent tooling) and is the standard answer to precisely this situation. It appears nowhere in the ADRs or the corpus, presumably because the corpus contains RFC 8439 but nothing newer on AEAD nonce handling.

---

### F-48 `[CORRECT]` "Zero-knowledge" is overstated in the same way twice.

ADR-020: *"The microservice cannot decrypt pointer_ciphertext… This is the zero-knowledge property: the service never sees plaintext data or decryption keys."*

True about *data*. But the pointer file's other contents — which 56 providers hold which chunk IDs for which `file_id` under which `owner_id` — are already in the microservice's own `chunk_assignments` table, because ADR-013 makes it the authoritative placement coordinator and ADR-002 needs it to dispatch audits.

So encrypting the pointer file protects the `file_key` and nothing else from the operator. The operator retains full placement metadata, file sizes, upload timing, and the owner↔file graph.

This is the same overstatement as F-11 (the HMAC DHT key), from a different direction. The property Vyomanaut actually has is **content confidentiality against the operator, with full metadata visibility to the operator**. That is a perfectly respectable position — Tahoe-LAFS is honest about the analogous limits — but it should be stated that way, particularly anywhere it reaches provider- or owner-facing copy.

---

### F-49 `[CORRECT]` ADR-021 Consequences still argues against its own Decision.

Decision: *"set max_hole_punch_retries = 1 (not the libp2p default of 3)."*
Consequences: *"DCUtR retry count of 3 adds 200–800 ms to hole-punch setup on every successful connection attempt, for only 2.4% additional gain. Reduce to 1 retry."*

The Consequences entry is the pre-decision argument left in place, now reading as though the retry count is still 3.

---

### F-50 `[COUNCIL]` Vyomanaut-operated relays sit on ~30% of the network's traffic and are unpriced and unmodelled.

ADR-021 Tier 3: symmetric-NAT providers use Circuit Relay v2, *"all traffic proxied,"* with *"relay nodes… Vyomanaut-operated and co-located in Indian cloud regions."* Paper 30 measures ~30% of P2P peers needing relay, and ADR-028 expects Indian CGNAT prevalence to *exceed* that baseline.

Three unexamined consequences:

1. **Cost.** Every byte of stored data, audit traffic, retrieval and repair for that ~30% transits operator-paid cloud egress. Indian cloud egress is roughly ₹7–9/GB. Under a repair burst — Qpeek ≈ 793 GB at N=1000 — the relayed fraction is a real, spiky bill that appears in no economic model. ADR-024's claim that *"repair bandwidth cost is borne by the receiving replacement provider, not by the microservice"* is false for the relayed third, which also undercuts F-41.

2. **Capacity.** ADR-029 sizes relay infrastructure at 3 nodes / 384 slots against *provider count* (90 relayed of 300). But slots are consumed per concurrent connection, and repair is the burst case — a correlated departure has many providers reconstructing simultaneously, each opening streams to 16 fragment holders. Headroom was computed against the steady state and the binding case is the burst.

3. **Position.** For that ~30%, the operator is on-path for all data-plane traffic. It cannot read content (AONT protects that) but it sees volumes, timing, and the full peer graph — and it is a single point of failure and a DoS target for a third of the network. This is a material qualification to "pure P2P data plane" and it compounds F-12.

---

### F-51 `[CORRECT]` ADR-016's idempotency key collides across event types.

`idempotency_key VARCHAR(64) UNIQUE`, specified as `SHA256(provider_id + audit_period)`. The uniqueness constraint is table-wide, but three event types share the table. A RELEASE and a later SEIZURE for the same `(provider_id, audit_period)` produce identical keys, and the second INSERT fails — silently blocking the seizure path ADR-024 §5 depends on.

Fix: include `event_type` in the digest.

---

## Corrections logged against earlier findings

**F-14 is partially resolved.** ADR-030 supplies the missing derivation: the ACTIVE transition requires `consecutive_audit_passes ≥ 80` **AND** `NOW() − first_chunk_assignment_at ≥ 120 days`. The duration comes from the 120-day floor, not the 80-audit threshold. ADR-005's prose should be amended to cite ADR-030 rather than implying the audit count produces 4–6 months. F-15 (the statistic measures audit reliability, not retention) is unaffected and still open.

**F-28 is confirmed and sharpened.** ADR-029 fixes the bootstrap gate at exactly ≥56 providers / ≥5 ASNs, which is the regime the durability model was never run against — and F-34 now shows the same gate is where confidentiality is weakest.

**F-02/F-03 gain a second, independent forcing reason.** F-40 shows daily full audit is unaffordable on the provider's disk, not only on the operator's database.

---

## What holds up

- **ADR-023** is the best ADR in the project. The write-amplification derivation is checkable, the single-writer goroutine invariant is stated in code rather than prose, the crash-recovery tail scan is correct, and the HDD compaction benchmark specifies a pass criterion and a tuning procedure. The F-40 objection is about a workload volume decided elsewhere, not about this engine.
- **ADR-027** found a genuine, subtle bug (per-replica secrets producing systematic false FAILs on failover) and fixed it properly, with a graceful rotation window matched to the audit cycle.
- **ADR-028** found another (cold-dial against a DHCP-rotated address) and fixed it with a mechanism whose cost is computed — 1.2 KB/day/provider.
- **ADR-029 Part B** — simulation mode as a day-one daemon feature with the readiness gate deliberately *not* bypassed, so tests remain valid proxies — is a genuinely good call that most projects make three years too late.
- **ADR-018's revision** is a model of how to update an ADR: it names the mechanism the previous version got wrong, explains why the papers changed the answer, and adds the migration cost it had previously omitted as a named trade-off.
- **ADR-016's addendum** correctly diagnoses that a hardcoded `held_earnings_paise = 0` was a *schema* gap, not a policy gap, and files against the right ADR.

---

## Routing summary

**To the council, in order:**

1. **F-32** — the audit verifies nothing. Ahead of everything else in both documents.
2. **F-34** — two-ASN plaintext recovery. Independent of F-32 and equally structural.
3. **F-35** — leader election for the six coordinated operations.
4. **F-36** — AONT cipher agility, before any production file is written.
5. **F-40 / F-41 / F-42** — audit I/O cost, seizure justification, vetting income stack. All three are provider-economics questions and should be taken in one session.
6. **F-44 / F-50** — V3 code family evidence; relay cost, capacity and position.
7. **F-33 / F-47** — vetting chunk derivation; nonce handling. Both are cheap fixes with a clear right answer; they need a ruling only because they change specs.

**Corrections (no ruling needed):** F-37, F-38, F-39, F-43, F-45, F-48, F-49, F-51.

**To the reading list:** F-34, F-35, F-40, F-43, F-44, F-46, F-47, F-50 → Domains K–O in `reading-list-v2-increment-2.md`.
