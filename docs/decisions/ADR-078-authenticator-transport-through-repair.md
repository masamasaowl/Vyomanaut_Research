# ADR-078 — Transport authenticators through repair

**Status:** Proposed — blocked on a proof check against Shacham & Waters Theorem 4.2, and on ADR-059's own F-01 council ruling
**Track:** LTS
**Topic:** #2 Proof of Storage · #4 Replication / Repair Protocol
**Supersedes:** — *(amends `ADR-059`'s field parameter; discharges its repair open constraint; reverses `ADR-076` §4's narrowing of Q68-3)*
**Superseded by:** —
**Research source:** Paper 74 (Chen, Ammula & Curtmola, CODASPY 2015 — `VerifyPartialRepairTag`'s scaled-tag property and the `GF(2^w)`-throughout construction), Paper 75 (Chen, Curtmola, Ateniese & Burns, CCSW 2010 — the proof of correct encoding), Paper 68 (Shacham & Waters — the primitive being amended)

---

## Context

`ADR-059` closes with the sentence this ADR exists to remove:

> **Repair breaks tagging, and nothing here fixes it.** A shard that *migrates* intact keeps valid
> authenticators — tags bind to block content and index, not provider identity. A shard that is
> *reconstructed* is new bytes at the same stripe position with no valid authenticators, and only
> the owner holds the keys to generate them, and the owner is offline by design. **A reconstructed
> shard is unauditable until this is answered.**

`Q68-3` states three shapes of answer and rejects all three. `ADR-076` §4 then removed the cheapest
of them — microservice-side tagging — on the grounds that after repair execution moves provider-side
the microservice *"holds neither the keys nor the bytes"* (it holds the keys; it no longer holds the
bytes), leaving a two-way choice it described as *"neither acceptable"* and making **R-47 the gating
read for the Proof of Storage milestone.**

R-47 is now read. It falsifies the premise underneath all of it.

### The premise, and why it is false

Every framing of Q68-3 assumes the new authenticator must be **computed**, and that computing it
requires one party to hold the secret key and the reconstructed content simultaneously. That is the
trilemma.

Paper 74 states the property that breaks it, in the course of explaining how its Aggregation Server
localises a faulty contributor:

> *"if `t_{i_j}` is the repair tag for the stored segment `b_{i_j}`, then `x·t_{i_j}` will be the
> repair tag for its partial segment `x·b_{i_j}`, considering both repair tags are generated based
> on the same key."* — Paper 74 §4.2

Paper 75 uses the same property five years earlier for a different job — a helper proves it applied
the right coefficients by returning `τ_i = Σ_j x_ij·T_ij`, the same linear combination applied to
its own tags, which the verifier checks without ever seeing the underlying blocks (Paper 75 §3.1.4).

**A tag that is linear in the block content, over the same field as the erasure code, travels through
the repair map with the data.** The new tag is not computed by anyone. It is transported. Nobody
exercises authority over it, so there is no authority to allocate, so there is no trilemma.

### Why it does not work today: one parameter

`ADR-059` evaluates authenticators in `Z_p`, `p = 2^128 + 51`, with each 16-byte sector read as a
big-endian integer. `internal/erasure` codes in `GF(2^8)` with irreducible polynomial `0x11d`.
Addition in `Z_p` carries; addition in `GF(2^8)` is XOR. The two operations do not commute, so

```
Σ_j z_j · (Σ_l α_l · m_{i_j,l})   ≠   Σ_l α_l · (Σ_j z_j · m_{i_j,l})
```

and the homomorphism does not transport. **This is F-LTS-14: `Q68-3` has stood open because of a
field choice made for codec convenience, in an ADR whose own justification for that choice
("byte-aligned 16-byte sectors, no rejection sampling") is fully preserved by the alternative.**

Paper 74 made the opposite choice deliberately and recorded the engineering cost of it: *"To achieve
an appropriate security level, we extended Jerasure's coding functions to support 128-bit symbols"*
(§1.2). They widened the **code** to reach the tag field. Vyomanaut does not have to: `8 | 128`, so
`GF(2^8)` is already a subfield of `GF(2^128)`.

### Why this is a new ADR and not an addendum

It amends `ADR-059`'s field parameter, discharges `ADR-059`'s repair constraint, reverses `ADR-076`
§4, and adds a protocol step to `ADR-076`'s elected-repairer design. It belongs wholly to neither
and touching only one would leave the other stating something false.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — reconstructed shards are unauditable** | No work | Repair is routine. Every repaired shard is permanently exempt from the audit that gates payment, scoring and further repair. `F-32` says the audit cannot currently fail; this makes a growing fraction of the network permanently exempt even after it can |
| **Give the repairing provider `(k_prf, α₁…α₆₄)`** | Trivially works; no field change | The repairer can forge passing proofs for the position it just wrote, forever. It is also a rotating party, so the key spreads across the provider population. Rejected in `Q68-3` and still rejected |
| **Leave shards unauditable until the owner returns** | No new mechanism | Unbounded. `ADR-004` makes the owner offline by design; durability cannot wait on a login |
| **Microservice receives reconstructed bytes and re-tags** | Cheap; microservice already holds the keys | Reverses `ADR-076` and reinstates `F-LTS-08` — the operator back on the plaintext path. Rejected |
| **`DPDP`-style authenticated update log (Paper 69)** | Makes whichever choice is made provable after the fact | Proves an update was *applied*, not that anyone was *entitled* to compute it. Wraps the trilemma; does not resolve it. Also requires transmitting real challenge indices, worse than the existing bandwidth problem at `ADR-060`'s volume |
| **Move the tag field to `GF(2^128)` and transport the tag — chosen** | Closes `Q68-3` constructively; no party holds key and plaintext together; tag size and storage overhead unchanged; supplies pollution-attack detection as a side effect; `ADR-059`'s byte-alignment rationale preserved and improved | Changes a parameter `ADR-059` declares frozen at first authenticator generation. Requires re-checking Shacham & Waters' security proof over a characteristic-2 field. Adds a protocol exchange to `ADR-076` |
| **Widen the erasure code to 128-bit symbols, as Paper 74 did** | Same result; matches the published precedent exactly | Unnecessary — the subfield relation makes it free. Would land the cost on RS encode throughput, the one quantity `Q65-1` already says is unmeasured |

## Decision

### 1. The authenticator field becomes `GF(2^128)`

Replace `ADR-059`'s

```
p = 2^128 + 51 = 340282366920938463463374607431768211507        (prime)
sector = 16 bytes, big-endian element of Z_p
```

with

```
F = GF(2^128), represented as GF(2^8)[Y] / (g(Y)),  g irreducible of degree 16 over GF(2^8)
    where GF(2^8) uses ADR-003's existing polynomial 0x11d
sector = 16 bytes (b_0 … b_15)  ↦  Σ_{i=0..15} b_i · Y^i
```

`s = 64` sectors per block, 1,024-byte blocks, 256 blocks per chunk, 4,096 B of authenticators per
chunk, **all unchanged.** Overhead stays at 1.5625%.

`ADR-059`'s stated reason for `p` — a byte-aligned 16-byte sector requiring no rejection sampling —
is preserved and strengthened: every 16-byte string is a valid `GF(2^128)` element unconditionally,
where `Z_p` required the bound `sector < p` to hold (it did, vacuously, but as a property to be
argued rather than one that cannot fail).

`g(Y)` must be pinned to a specific polynomial in `internal/audit` and covered by a
compiler-visible guard in the same style as `TestProfileShardSizeIsConstant`. **The basis is part of
the wire format**, not an implementation detail: a different basis produces different tags for the
same bytes.

### 2. Why the subfield embedding makes the transport exact

`GF(2^128)` is a 16-dimensional vector space over `GF(2^8)` with basis `1, Y, …, Y¹⁵`. For
`z ∈ GF(2^8)` with embedding `ι(z)`, multiplication `ι(z) · Σ b_i Y^i = Σ (z·b_i) Y^i` — by
definition of scalar multiplication in a vector space over `GF(2^8)`.

That is **exactly** what `internal/erasure` does: bytewise `GF(2^8)` multiplication of each of the
16 bytes of a sector. **The RS scaling operation and `GF(2^128)` scalar multiplication by an
embedded subfield element are the same operation.**

### 3. The transport identity

For target shard `y` reconstructed from a 16-subset `S` with RS coefficients `z_j ∈ GF(2^8)`, block
index `i_y` at the target position and `i_j` at helper `j`'s corresponding position:

```
σ_y  =  Σ_{j∈S} ι(z_j)·σ_{i_j}   +   Δ

           Δ  =  f_kprf(i_y)  +  Σ_{j∈S} ι(z_j)·f_kprf(i_j)
```

(In characteristic 2, `+` and `−` coincide.) `Δ` is a function of `(k_prf, {i_j}, i_y, {z_j})` and
**contains no chunk data of any kind.**

### 4. The repair-time protocol, as an addition to ADR-076

`ADR-076` puts execution provider-side and keeps the microservice as elector and bookkeeper. Extend
its elected-repairer protocol with:

1. **Helper `j` contributes a pair:** `z_j ⊙ shard_j` (the existing shard contribution) **and**
   `ι(z_j)·σ_j` — 4,096 B of scaled authenticators per chunk, `+1.5625%` on the shard contribution.
2. **The repairer verifies every contribution before aggregating.** Each pair
   `(z_j ⊙ shard_j, ι(z_j)·σ_j)` is checked by `ADR-059`'s verification equation applied to the
   scaled pair. **This is mandatory, not optional** — see §5.
3. **The repairer aggregates:** `shard_y = Σ_j z_j ⊙ shard_j` and `σ' = Σ_j ι(z_j)·σ_{i_j}`.
4. **The repairer sends `σ'` to the microservice** — 4,096 B per chunk, no shard bytes.
5. **The microservice returns `σ_y = σ' + Δ`**, computing `Δ` from key material and metadata alone.
6. **The repairer writes `(shard_y, σ_y)` to the replacement provider** under the existing
   `ADR-072` capability token.

**`Δ` must never be transmitted.** The microservice returns the completed `σ_y`, not the correction.
Publishing `Δ` would hand out one linear equation in 17 PRF outputs per repaired block, accumulating
across repair events over overlapping index sets. Whether that system becomes solvable at realistic
repair volumes has not been derived, and the protocol should not require the answer.

**The microservice receives no shard bytes at any step.** `ADR-076`'s central property — the
operator leaves the plaintext path — holds unchanged.

### 5. Per-contribution verification is mandatory

Paper 75 §3.1 names the **pollution attack**: a helper that behaves honestly during Challenge and
contributes corrupted bytes during Repair. MDS reconstruction from `k = 16` shards will produce a
well-formed shard from one corrupt input, so *"it decoded"* is not evidence. `ADR-076`'s protocol as
specified has no helper-verification step at all, and without one a single malicious helper writes a
permanently and undetectably wrong shard that consumes one of the 40 tolerated fragment losses.

The check binds in both directions. A helper contributing corrupt data with the *real* tag produces
an inconsistent pair, caught immediately. A helper wanting a self-consistent forgery needs
`(k_prf, α₁…α₆₄)` to compute a tag for its corrupted bytes, which it does not hold.

Cost: 4,096 PRF evaluations and 262,144 field multiplications per repaired chunk across all 16
helpers — 0.56% of one provider-day's routine verification load.

### 6. Where verification runs

**On the repairer**, by default. It is cheap, it is self-serving (a repairer has no incentive to
accept corrupt input), and it keeps 65,536 B/chunk of tag traffic off the microservice. The residual
is that a repairer cannot be *proved* to have run it. Moving the check to the microservice costs
65,536 B per chunk — 2.1 MB per 32-fragment gated repair event — and buys accountability. **Not
decided here.** → Q75-1.

### 7. Order of work

**`ADR-076` §1 first.** The cost arguments in this ADR are priced against `r0 = 8`-gated repair.
Under the eager repair the system currently performs (`F-LTS-07`), `Δ` generation and
per-contribution verification scale with churn rather than with durability events, and no number
here holds.

## Consequences

**Positive:**

- **`Q68-3` closes.** A reconstructed shard carries a valid authenticator from the moment it is
  written. Repaired shards stop being permanently exempt from the audit that gates payment, scoring,
  and further repair.
- **No party holds the key and the plaintext together.** Helpers hold shard bytes and no keys; the
  microservice holds keys and no bytes. The trilemma is not resolved by choosing a lesser evil; it
  does not arise.
- **`ADR-076` §4 is reversed cleanly.** The option it closed returns in a form that satisfies the
  constraint that closed it, because `Δ` is not in the data path.
- **Pollution-attack detection arrives with it**, on grounds entirely independent of tag continuity
  — a gap `ADR-076` has today and does not name.
- **Zero storage cost.** Tag size, block layout, chunk overhead, and the 1,040-byte audit response
  are all unchanged.
- **`ADR-059`'s byte-alignment rationale is preserved and improved.** No rejection sampling, no
  bound to argue.
- **`PCLMULQDQ` is universal on target hardware** — the same carry-less multiply GCM uses. On x86-64
  with that instruction, `GF(2^128)` multiplication is likely *cheaper* than 128-bit modular
  multiplication, not more expensive.
- **`R-47` discharges as the gating read for the Proof of Storage milestone**, per `build_part4.md`.

**Negative / trade-offs:**

- **A security proof must be re-checked, not assumed.** Shacham & Waters state the private scheme
  over `Z_p`. The argument is linear algebra over a field and should carry at `2⁻¹²⁸`, and both
  Paper 74 (`GF(2^w)`) and Paper 75 (`GF(p)`) instantiate Shacham–Waters-style tags in other fields
  — but characteristic 2 is a real difference and any step relying on `x + x ≠ 0`, on division by 2,
  or on integer ordering fails. **This is the blocking constraint.**
- **`ADR-059` declares `(p, s)` frozen at first authenticator generation.** No production
  authenticators exist, so the change is free today and impossible later. It must land before the
  Proof of Storage milestone writes a tag in any environment that outlives a database reset.
- **A new coupling between `ADR-003` and `ADR-059`.** The code field must divide 128. `GF(2^8)`
  does; `GF(2^16)` (LESS's requirement, per `Q65-1`) also does. That is a coincidence, and it should
  be recorded as a constraint rather than relied on as a property.
- **`g(Y)` becomes wire format.** A different irreducible polynomial or basis ordering produces
  different tags for identical bytes. It joins `ShardSize` as a compile-time constant with a guard.
- **A new protocol surface on `ADR-076`'s elected-repairer exchange**, with its own authentication
  and failure modes: a repairer that requests `Δ` for a repair it is not conducting, or for an index
  set it did not receive contributions for.
- **The elected repairer still obtains plaintext.** `F-69` is untouched. This ADR closes the
  *auditability* half of the repair problem and does nothing for the *confidentiality* half.

**Open constraints:**

- **Shacham & Waters Theorem 4.2 over `GF(2^128)`.** Blocking. → Q78-1.
- **`internal/erasure` reconstruction must be a pure `GF(2^8)`-linear map** of surviving shards with
  no special-cased path. Verify `invertMatrix` and the Vandermonde construction in
  `rs_internal.go`; the systematic first `k` positions give a different coefficient vector but
  linearity holds. Any non-linear fallback path breaks this ADR for that path.
- **`Δ` must not be transmitted.** Enforced at the protocol boundary, not by convention.
- **Where per-contribution verification runs** is undecided. → Q75-1.
- **`ADR-076` §1's `r0` gate must be built first**, or the cost argument does not hold.
- **`F-LTS-11` is unaffected and still blocks.** Where the 4,096 B of authenticators live relative to
  IC §4.1's fixed 262,252-byte Frame 1 is unchanged by this ADR — the tags are the same size in the
  same place. This ADR adds a second consumer of that decision: the repair path now also ships
  authenticators on the wire.
- **The demo is unaffected and stays unaffected.** `ADR-062` freezes it at `demo-v1.0.0`. `Track:
  LTS`; must not be backported.

## References

- [Paper 74 — Chen, Ammula & Curtmola, RDC-EC](../research/paper-74-chen-ammula-curtmola-rdc-ec.md): the scaled-tag property; the `GF(2^w)`-throughout construction and its 128-bit-symbol extension
- [Paper 75 — Chen, Curtmola, Ateniese & Burns, RDC-NC](../research/paper-75-chen-curtmola-ateniese-burns-rdc-nc.md): the proof of correct encoding; the pollution attack §5 defends against
- [Paper 68 — Shacham & Waters, Compact PoR](../research/paper-68-shacham-waters-compact-por.md): the primitive amended; Theorem 4.2 is the proof obligation
- [Paper 69 — Erway et al., DPDP](../research/paper-69-erway-dpdp.md): the option this supersedes; DPDP proves an update was applied, not who was entitled to compute it
- [ADR-059 — Homomorphic authenticator audit](ADR-059-homomorphic-authenticator-audit.md): field parameter amended; repair open constraint discharged
- [ADR-060 — Sampled chunk audit schedule](ADR-060-sampled-chunk-audit-schedule.md): unaffected; sampling rule and rate unchanged
- [ADR-076 — Repair topology](ADR-076-repair-topology.md): §4's Q68-3 narrowing reversed; elected-repairer protocol extended
- [ADR-003 — Erasure coding](ADR-003-erasure-coding.md): the `GF(2^8)` field is now also an audit-primitive parameter
- [ADR-072 Addendum A — Position binding](ADR-072-addendum-a-position-binding.md): the capability token the repairer reuses to write
- [ADR-019 — Zero-knowledge storage](ADR-019-zero-knowledge-storage.md): unchanged; the microservice still receives no plaintext
