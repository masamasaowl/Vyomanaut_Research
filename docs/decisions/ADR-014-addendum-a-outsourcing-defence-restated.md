# ADR-014 Addendum A — Defence 2 is stale and unparameterised

**Status:** Proposed — blocked on a measurement that does not exist (see §4)
**Track:** LTS
**Topic:** #19 Adversarial Provider Behaviour · #2 Proof of Storage
**Amends:** ADR-014 Defence 2 (outsourcing attack). Defences 1, 3, 4 and 5 are unchanged.
**Research source:** Paper 73 (Chen & Curtmola, J. Computer Security 25(6) 2017 — the BDS-model measurement, the α-cheating adversary, and Theorem 3.2), Paper 37 (SHELBY, re-read — Observation 1's granularity footnote)

---

## Context

ADR-014 Defence 2 defeats the outsourcing attack — a provider that fetches data on demand instead of
storing it — with a response deadline:

```
deadline = (chunk_size / p95_measured_upload_throughput) × 1.5
         = (256 KB / 500 KB/s) × 1.5 = 768 ms
```

plus a latency floor at `0.3×` the same quantity, flagging anomalously fast responses as
just-in-time retrieval. It cites Paper 37's Observation 1 as its justification.

Two things have happened since. ADR-059 and ADR-060 replaced the audit wire format the deadline is
computed against, and Paper 73 was read.

### F-LTS-13 — the formula's inputs no longer exist

Under ADR-059 the audit response is **1,040 bytes, constant, regardless of how many chunks were
sampled.** Under ADR-060 there is **one challenge per `(provider, file, day)`** covering 2,867
sampled chunks, each read whole.

So `chunk_size / p95_upload_throughput` is a ratio with no referent. The wire carries 1,040 bytes in
every case; upload throughput bounds nothing. Meanwhile ADR-060's own accounting puts the honest
prover's work at **0.752 GB of disk reads taking 36.19 s**. A deadline of 768 ms against a 36-second
honest path rejects every honest provider by a factor of roughly 47. The latency floor is worse: it
flags as suspicious any response faster than 230 ms, which is *every* response, since the 1,040-byte
reply is not throughput-bound at all.

Defence 2 is not merely imprecise. **As specified against the current audit format it is
inoperable.** No implementation can have been following it, which means either the deadline is not
enforced or it is enforced against something the ADR does not describe.

### F-LTS-12 — ADR-060's detection bound is conditional on a deadline that binds

ADR-060 states that at 1% sampling a 1% deletion is detected with probability `1 − 4.0 × 10⁻¹³`.
That is Ateniese's bound and it is correctly derived — but it bounds the probability that the
**sample misses the deleted chunks**, not the probability that the prover **cannot answer**.

In a distributed erasure-coded system those are different events. A provider that deleted a chunk
can obtain it: fetch the 16 peer shards at the same segment offset and re-encode. The cryptographic
response is then perfectly valid, because ADR-059's response is a pure function of the challenged
block contents and the seed, and nothing binds it to *when* the bytes arrived.

**The only thing separating a provider that stored the data from one that fetched it is elapsed
time.** ADR-060's detection table therefore rests entirely on Defence 2, and ADR-060 does not
mention Defence 2 anywhere.

### The measurement that says timing-only arguments are fragile

Paper 73 §2.1 measured the Benson–Dowsley–Shacham model — establishing possession purely by response
deadline — against a real provider. Its central assumption (no fast private path between servers) is
false on AWS: inter-datacentre bandwidth 11–36 MB/s against under 1 MB/s from outside, and 11 ms
propagation between two of the regions. Substituting into the model's own inequality gives an
admissible challenge of **2.66 blocks**, against the **460** Ateniese requires for 99% confidence at
1% corruption.

Defence 2 is a BDS-shaped argument. Vyomanaut's providers being consumer machines rather than
datacentres does not rescue it: a colluding set on the same ISP has exactly the fast private path
the model assumes away, and Paper 73's adversary is explicitly a colluding set.

### Theorem 3.2 — the deadline is sized against the wrong adversary

Paper 73 §3.2.2 defines the **α-cheating adversary**: a provider meeting its contract with only an
`α` fraction of the required storage. Theorem 3.2 proves its optimal strategy is to store an equal
`α` fraction **everywhere**, not full copies in some places and nothing in others — so a challenged
provider already holds `α` of what is asked and must fetch only `(1 − α)`.

ADR-014 Defence 2 has no `α` anywhere. It implicitly assumes `α = 0` — a total cheat — which
Theorem 3.2 says is the adversary's *worst* available strategy and therefore never its chosen one.

### The substitution

Honest path, from ADR-060's own figures:

```
honest bytes = 2,867 × 262,144 B = 0.752 GB
honest time  = 2,867 × (10 ms seek + 262,144 B / 100 MB/s) = 2,867 × 12.62144 ms = 36.19 s
```

Full outsourcing (`α = 0`) requires `k = 16` peer shards per sampled chunk:

```
ROTF bytes = 2,867 × 16 × 262,144 B = 12.03 GB
```

| Provider downlink | ROTF transfer | Ratio to honest 36.19 s |
| --- | --- | --- |
| 10 Mbit/s | 2.67 h | **266×** |
| 50 Mbit/s | 0.53 h | **53×** |
| 100 Mbit/s | 16.0 min | **27×** |
| 300 Mbit/s | 5.4 min | **8.9×** |

Against a total cheat the margin is enormous and Defence 2's intent is sound. Now the α-cheater.
Timing detects the cheat only when it takes longer than honest:

```
(1 − α)·c·k·L / B   >   c·(seek + L/D)
(1 − α)             >   B · 0.01262144 / (16 × 262,144)
(1 − α)             >   B × 3.00918 × 10⁻⁹            B in bytes/s
```

| Provider downlink | Deletable fraction invisible to timing |
| --- | --- |
| 10 Mbit/s | **0.376%** |
| 25 Mbit/s | **0.940%** |
| 50 Mbit/s | **1.881%** |
| 100 Mbit/s | **3.762%** |
| 300 Mbit/s | **11.28%** |

*Hand check at 100 Mbit/s:* honest reads 0.752 GB in 36.19 s; the invisible cheat fetches
`0.03762 × 12.03 GB = 0.4525 GB` at 12.5 MB/s = 36.2 s. Equal, as the crossover requires. ✓

The model ignores peer upload limits, RTT, relay overhead and RS decode CPU — all of which help the
defender. It is the attacker-favourable direction, which is the correct direction for a security
bound.

**At 50 Mbit/s and above, a 1% deletion — the exact case ADR-060 claims to detect with probability
`1 − 4.0 × 10⁻¹³` — is invisible to timing.** At 300 Mbit/s a provider can delete 11% of its
holding and the audit reports perfect health.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Leave Defence 2 as written** | No work | It is inoperable against the current wire format, and ADR-060's detection claim silently depends on it |
| **Delete Defence 2, rely on Observation 1's cost argument alone** | Honest about what timing can carry; Paper 37 Observation 1 holds at Vyomanaut's parameters with a large margin | Observation 1 is a *monthly-epoch cost* argument. It rules out a permanent outsourcing business model; it says nothing about an opportunistic α-cheat inside one audit. Paper 73's whole point |
| **Tighten the deadline to catch a 1% α-cheat** | Restores the property on paper | Requires setting `τ` within a few percent of the honest median on a consumer desktop doing 2,867 random seeks under ADR-025's background gate, competing with the user's workload and storage-engine compaction. Paper 73 §4.2 measured a 1.36× spread at the 95th percentile *in a datacentre* and warns that benign tail latency destroys separation. No distribution exists here |
| **Restate the deadline correctly, declare it a secondary signal, and name what it cannot carry — chosen** | Removes a formula that cannot be implemented; makes ADR-060's dependency explicit; states the adversary model; stops the deadline being cited as a primary defence it cannot be | Leaves a real, unclosed gap at the α-cheating adversary. Does not fix it |
| **Bind the response to a provider-local non-transferable secret** | Would close it by primitive rather than deadline | No construction exists for the erasure-coded case in Papers 66–75. This is a research item, not an option |

## Decision

### 1. Defence 2's formula is withdrawn and restated

The per-chunk deadline, the 768 ms worked example, and the `0.3×` latency floor are **withdrawn**.
They are computed against a per-chunk challenge/response that no longer exists.

The deadline is restated in the form Paper 73 §4.2 derives:

```
τ  =  c · x  +  2·t_i

  c    = chunks sampled in this audit                        (ADR-060: 2,867 at a 70 GB provider)
  x    = measured per-chunk honest contribution
         = seek + (262,144 B / sequential read rate) + ADR-059 field arithmetic per chunk
  t_i  = microservice↔provider network delay
```

`τ` is computed **per audit**, from the audit's own `c`, not per chunk. It scales with the sample,
which is the quantity that actually varies.

`x` must come from **measurement of the honest population**, per provider, not from a nominal disk
figure. `p95_throughput_kbps` in the reliability DB is repurposed: it stops meaning upload
throughput (which no longer bounds anything) and becomes the provider's measured per-chunk honest
audit contribution, maintained by the same EWMA ADR-006 uses.

### 2. Defence 2 is demoted to a secondary signal

The primary deterrent against outsourcing is the **reconstruction cost imposed by RS(16,56)** — 16
peer shards per chunk, 12.03 GB per full audit — which is Paper 37 Observation 1 evaluated at
Vyomanaut's parameters and holds with a 27–266× margin against a total cheat. ADR-014's framing of
Defence 2 as *"use Filecoin's Seal concept"* overstates what a deadline provides and should be read
as what it is: a timing check layered on top of a cost argument.

Anomaly handling (`jit_flag`, the 7-day window, the 0.5× scoring weight, the collusion escalation on
identical latencies) is **retained unchanged**. It was always a statistical signal rather than a
gate, and it is the part of Defence 2 that still does its job.

### 3. The adversary model is stated

Defence 2 is sized against an **α-cheating adversary** (Paper 73 §3.2.2) with `α` stated explicitly
as a design parameter, not against a total cheat. `α` must be chosen and written into the
`NetworkProfile` alongside `audit_sample_rate`, with the same compiler-enforced relationship to the
durability target that ADR-060's open constraints already require. **A defence with no adversary
parameter cannot be evaluated, and ADR-014 currently has none for any of its five classes.**

### 4. What is recorded as unclosed

**At the currently plausible range of Indian consumer downlink, timing does not separate an
α-cheater from an honest provider at the deletion fractions ADR-060 claims to detect.** This ADR does
not fix that. It records it, and names the three things that would:

- **A measurement.** The honest per-chunk distribution `x` on real provider hardware, and the median
  and tail of provider downlink. Both are LaunchGate items; the first shares scope with Q23-1's
  RocksDB rate-limiter measurement and Q65-1's RS throughput work. → Q73-2.
- **A structural check.** Whether a provider can obtain another provider's shard bytes at all.
  ADR-072 capability tokens gate authorised download, but nothing prevents colluding providers
  serving each other over libp2p off-protocol. If the answer is that they can, the finding stands as
  written; if there is an enforced barrier, the attack costs collusion rather than bandwidth. →
  Q73-1.
- **A primitive.** A response bound to something non-transferable and provider-local. Not available
  in Papers 66–75 for the erasure-coded case. → Domain A.

### 5. ADR-060 gains an explicit dependency

ADR-060's detection table is a bound on **possession**, not on **ability to answer**. Its open
constraints must record that the bound holds only while `τ` binds, and that at `α` close to 1 it
does not. ADR-060's sampling rule and its chosen 1% rate are **unaffected** — the sampling design is
right; the claim made about what it detects is conditional in a way the ADR does not say.

## Consequences

**Positive:**

- A formula that cannot be implemented against the current wire format is removed rather than left
  to be discovered at implementation time.
- ADR-060's detection claim stops being unconditional. The condition is named and its failure region
  is quantified.
- The outsourcing defence acquires an adversary model, which is the first of ADR-014's five classes
  to have one.
- The reconstruction-cost argument is promoted to primary, which is both where the real margin is
  and what Paper 37 actually proves.
- `p95_throughput_kbps` acquires a meaning that matches what it now measures.

**Negative / trade-offs:**

- A real gap is left open. At 100 Mbit/s a 3.76% deletion is timing-invisible; ADR-060 says a 1%
  deletion is caught with near-certainty. Both statements are true and they are about different
  events.
- `τ` cannot be set until `x` is measured, and no measurement exists. Until then any value is
  `[VENDOR-DEFAULT]`-class and must be tagged as such per ADR-077's triggers.
- Provider-facing copy (ADR-045) inherits this. ADR-060 already requires the probabilistic nature of
  sampling to appear there; it must not be described in a way that implies detection of a determined
  α-cheater.

**Open constraints:**

- `x` and the provider downlink distribution are unmeasured. → Q73-2, LaunchGate.
- Whether providers can serve each other shard bytes off-protocol. → Q73-1.
- `α` must be chosen and profiled. Until it is, §3 is a shape, not a parameter.
- Whether any implementation in `internal/audit` computes a per-chunk deadline. If it does, the
  divergence between spec and code is a separate finding.
- Paper 37 Observation 1's validity depends on full-chunk reads (its footnote 6). ADR-060 enforces
  that. Any future range-read optimisation on the retrieve path must exclude the audit path
  explicitly, or the primary defence weakens by the sampling ratio. → ADR-060 Addendum A.
- **Demo unaffected.** `ADR-062` freezes it; `Track: LTS`; not backported.

## References

- [Paper 73 — Chen & Curtmola, RDC with server-side repair](../research/paper-73-chen-curtmola-rdc-sr.md): the BDS measurement, the α-cheating adversary, Theorem 3.2, and `τ = c·x + 2·t_i`
- [Paper 37 — SHELBY](../research/paper-37-shelby-incentive-compatibility.md): Observation 1 and its chunk-granularity condition; revised this session
- [ADR-014 — Adversarial defences](ADR-014-adversarial-defences.md): Defence 2 amended; Defences 1, 3, 4, 5 unchanged
- [ADR-059 — Homomorphic authenticator audit](ADR-059-homomorphic-authenticator-audit.md): the 1,040-byte constant response that invalidates the old formula
- [ADR-060 — Sampled chunk audit schedule](ADR-060-sampled-chunk-audit-schedule.md): gains an explicit dependency on `τ`; sampling rule unchanged
- [ADR-060 Addendum A](ADR-060-addendum-a-shelby-substitution-corrected.md): the companion correction
- [ADR-005 — Peer selection](ADR-005-peer-selection.md): the vetting period that measures `x`
- [ADR-017 — Audit receipt schema](ADR-017-audit-receipt-schema.md): `response_latency_ms` and `jit_flag` retained
- [ADR-077 — Research-first triage](ADR-077-research-first-triage.md): `τ` is a `[UNDERIVED]` parameter until `x` is measured
