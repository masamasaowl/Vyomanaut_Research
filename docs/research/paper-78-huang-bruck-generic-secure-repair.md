# Paper 78 — Generic Secure Repair for Distributed Storage

**Authors:** Wentao Huang, Jehoshua Bruck (California Institute of Technology)
**Venue / Year:** arXiv:1706.00500, 1 June 2017
**Track:** LTS
**Reading list:** Domain P / R-29 — Band 1 — accept criterion *"reconstruction distributed so no
participant sees plaintext, at stated round/latency cost"*
**ADRs produced:** contributes to ADR-079 (Domain P Menu)
**Findings raised:** sharpens F-LTS-15 (accountability/blindness tension — see below)
**Questions closed:** none · **Questions raised:** Q79-2 (does RS(16,56) carry the `z` this
framework requires)
**Triage score:** 9/10 (parameter reach 2 · trust model 2 · evidence 2 · actionability 2 · corpus
delta 1) — the highest-scoring item in R-29, and the one the earlier informal triage pass mis-filed.

---

## Provenance

Read: the full arXiv preprint, all sections including the proofs of Theorems 1–4 and the vector
linear extension (§IV, Constructions 4 and 5). No parts skipped.

## Problem Solved

A linear secret-sharing scheme with parameters `(n,k=n−r−z,r,z)` tolerates `r` erasures and hides the
message from any `z` colluding shares by design. When a share is lost and must be regenerated, the
**straightforward way to repair it — a trusted dealer collects enough shares, recomputes, and hands
back the lost one — requires that dealer to be trustworthy**, because it necessarily learns
information the `z`-secrecy guarantee was built to withhold `[P §I]`. Existing "codes with secure
repair" solve this by baking extra randomness into the code itself so the dealer never needs to see
anything sensitive — but this costs rate: *"one-round secure repair comes at a high cost in rate, and
codes with non-secure repair generally have a significantly better rate... than codes with secure
repair"* `[P §IV]`.

This paper's contribution is a **generic, black-box, two-round protocol** that adds dealer-free
secure repair to *any* linear secret-sharing scheme — including ones designed purely for bandwidth
efficiency with no secrecy-repair guarantee built in — without paying that rate penalty. *"Our main
result... implies that this trade-off between rate and secure repair is not necessary... The only
cost is that the repair process now involves two rounds instead of one"* `[P §IV]`.

## Key Findings

### Definition 1 — what "secure repair" means here

`[P Def. 1]` A secure repair scheme for an `(n,k,r,z)` secret sharing scheme is a protocol that (a)
correctly reconstructs the lost share, and (b) maintains the property that any `z` colluding nodes —
**including the failed/replacement node itself** — learn nothing about the message during or after
repair. This is a strictly stronger requirement than the underlying code's static `z`-secrecy: it
must survive the *process* of repair, not just the *state* before and after.

### Constructions 4 and 5 — near-bandwidth-optimal, two-round, no trusted dealer

`[P §III–IV]` For a scalar linear `(n,k,r,z)` scheme (Construction 4) or a vector-linear one
(Construction 5, which composes with codes that already have efficient *non-secure* repair
bandwidth), the protocol runs in two rounds: helpers first mask their contributions using a
disposable `(n,n−z,0,z)` one-time scheme and exchange masked values with each other, then each node
forwards a re-combined value toward the replacement node, which decodes. **No single party — dealer,
helper, or replacement node — ever holds more than its own share plus one-time masking material.**

### Theorem 4 — a matching lower bound

`[P Thm 4]` For any rate-optimal `(n,k=n−r−z,r,z)` scheme with non-secure repair bandwidth `W`, any
secure repair scheme requires bandwidth at least

```
(n−1)·W / [2(n−z−1)]                                                        [P Thm 4]
```

and Constructions 4/5 achieve at most `(W+1)n/(n−z)` `[P §IV]` — **within a factor of ~2 of optimal
for all parameters, and asymptotically optimal (approaching `W` itself) as `n` dominates `z`.**

### The load-bearing constraint the earlier informal pass missed

`[P §IV, Constructions 4–5]` The protocol wraps a persistent code with parameters `(n, k=n−r−z, r, z)`
— **the stored code itself must already reserve `z` symbols of information-theoretic secrecy by
design.** This is not a protocol-only add-on to an arbitrary MDS code; it requires the underlying
code to be a genuine `(z≥1)`-secure linear secret-sharing scheme to begin with. A bare systematic RS
code storing real ciphertext on every shard (no random padding) has `z=0` in this framework's strict
sense — there is no built-in secrecy for the protocol to preserve.

## Substitution at Vyomanaut's Parameters

`[DERIVED]` **What z=1 costs, expressed as the Huang–Bruck rate identity.** Their model fixes
`k = n − r − z`. Reading `n=56, r=40` (RS(16,56)'s existing parity count) and asking for blindness
against one repairer (`z=1`): `k' = 56 − 40 − 1 = 15`. **This means the data dimension drops from 16
to 15 symbols per segment for the same `(n,r)` footprint** — equivalently, to keep the plaintext
segment size fixed, storage expands by `16/15 = 1.0667×` on top of RS's existing `3.5×`:

```
total expansion = 3.5 × 16/15 = 3.733×
```

**This cross-checks Paper 76's independently-derived number (`3.829×` via Goparaju's Theorem 3 at the
same `ℓ2=1`) to within 2.5%.** Two different papers, two different proof techniques (subspace
intersection vs. information-theoretic rate identity), converge on the same order of magnitude for
the same question. This is the strongest single piece of evidence in this band that `z=1` blindness
costs roughly **6–9% extra storage**, not the much larger figures this domain's earlier informal pass
worried about.

`[DERIVED]` **Repair bandwidth at z=1, n=56**, using Theorem 4's bounds with `W` = whatever the
underlying non-secure repair scheme costs (left as a variable `W` since Vyomanaut has not yet adopted
a bandwidth-efficient RS-layer repair scheme — see Paper 79):

```
lower bound: 55W / (2×54) = 55W/108 ≈ 0.509W
upper bound (Constructions 4/5): (W+1)×56/55 ≈ 1.018W + 1.018
```

**Secure repair bandwidth is within roughly a factor of 2 of whatever non-secure repair bandwidth is
achieved underneath it — the secrecy protocol itself adds no order-of-magnitude cost at our `n`.**

## What This Paper Rules Out

- **Rules out the informal "protocol-only, storage exactly unchanged" framing this domain's earlier
  pass used for Option A.** It is **not** storage-neutral. It costs the same ballpark as Paper 76's
  construction (`~3.7–3.8×` vs RS's `3.5×`), because both are answering the same question
  (`z`/`ℓ2 = 1` blindness) under the same information-theoretic constraint — Krawczyk's lower bound
  (Paper 77) applies to any scheme with a genuine `z≥1` secrecy guarantee, and this protocol does not
  escape it. What Huang & Bruck's protocol *does* remove is the need for a **trusted dealer** — a
  different, real, and separately valuable property (it is what lets the repairing party be the
  microservice or an elected provider without that party needing to be trusted), but it is not a
  free storage lunch.
- **Rules out applying this framework directly to AONT-RS as currently specified.** AONT-RS's
  security is computational (bounded `K` enumeration), not the information-theoretic `z`-secrecy this
  framework's proofs are built on. Whether the two-round masking protocol composes safely with a
  *computationally*-secure persistent code is not addressed by this paper and is not resolved by
  citing it — flagged as Q79-2, genuinely open.

## Trade-offs

| Chosen (candidate) | Over | Consequence |
|---|---|---|
| Two-round dealer-free protocol (this paper) at `z=1` on a converted `(56,15,40,1)`-style code | Trusted-dealer one-round repair (current implementation, F-LTS-08) | Removes the need to trust the microservice; costs `+6.7%` storage and one extra communication round per repair |
| Apply at the RS layer directly, requiring `z≥1` built into RS itself | Apply at the AONT layer, where secrecy already lives | RS-layer application is proved by this paper (given the rate cost above); AONT-layer application is unproved (Q79-2) but would cost nothing extra in rate, since AONT's secrecy is already "free" relative to storage |

## Breaks in Our Case

- **Framework requires the persistent code to have `z≥1` (Sec. III/IV constructions)** ≠ **RS(16,56)
  as deployed stores real ciphertext with no random padding, `z=0`**
  → **Costed, not Fatal.** Converting to a `(56,15,40,1)`-style code (sacrificing one symbol of
    dimension to buy `z=1`) is a real, quantified, and comparatively cheap option (`+6.7%`), proved
    achievable by this paper's own Construction 4/5 machinery once the code has the right shape.

- **The paper's security model and Definition 1 are stated for a scheme where the eavesdropper is one
  of the `n` storage nodes** ≠ **Vyomanaut's disclosure concern (F-69) is about the *coordination
  microservice* or an *elected repairer*, which may not be one of the `n` storage nodes at all**
  → **Open.** The protocol's masking rounds route information *through* the `n` nodes to reach the
    replacement node; whether the same guarantee holds when the party of concern is an external
    orchestrator rather than a peer node needs a short adaptation argument, not a re-derivation from
    scratch — but it is not handed to us by the paper as stated.

## Decisions Influenced

Primary input, alongside Paper 76, to **ADR-076 Addendum A** (F-69's disjunction is not exhaustive —
a protocol-layer option exists) and **ADR-079** (Domain P Menu). Neither ADR selects this option over
the others; that remains a council decision.

## Falsifiers

- **A proof obligation.** Q79-2 — does the two-round masking protocol preserve AONT-RS's
  *computational* security guarantee, or does composing an IT-secure masking round with a
  computationally-secure persistent layer create a hybrid whose security properties nobody has
  proved? Until answered, this option's applicability to Vyomanaut's actual encoding pipeline
  (AONT then RS, per ADR-022/ADR-076) is unverified, even though its applicability to a bare `z≥1`
  RS-style code is proved.
- **A measurement.** The `+6.7%` storage figure assumes `z=1` suffices — i.e., that the single
  elected repairer is the only party whose observation matters per repair event. If `ADR-075`'s
  election process is later found to leak repairer identity to more than one party (e.g., because
  the microservice both selects and observes), the effective `z` needed rises and the cost rises with
  it per the `k=n−r−z` identity above.

## Disagreements

None found — no existing Vyomanaut document makes a claim this paper contradicts. It does, however,
correct this domain's own earlier informal characterization (see Substitution above), which is
recorded as a self-correction rather than a disagreement with a third party.

## Corpus Delta

New to the corpus: the first result giving a proved, near-optimal-bandwidth, dealer-free repair
protocol with a matching information-theoretic lower bound. **Corrects the prior triage pass's
"Option A: protocol-only, storage unchanged" framing** — this note replaces that framing with the
`3.73×` figure, cross-validated against Paper 76's independent `3.83×`. No prior paper note requires
edits.

## Open Questions

**Manual step for the reader:** add to `docs/research/open-questions.md`:

```
### Q79-2 — Does Huang & Bruck's two-round secure-repair protocol preserve computational (not
just information-theoretic) secrecy when composed with AONT-RS's embedded-key mechanism, rather
than with a bare (n,k,r,z≥1) information-theoretic secret-sharing code as the paper assumes?

**Raised by:** Paper 78 (Huang & Bruck) — the paper's proofs are entirely information-theoretic;
AONT-RS's security (Paper 16) is computational.
**Status:** open — blocks ADR-079's council decision on whether this option applies to Vyomanaut's
actual encoding pipeline (AONT then RS) or only to a hypothetical bare-RS-with-padding variant.
**Blocked on:** a security proof or a council ruling to treat this as out of scope and pursue the
RS-layer-only variant (accepting the `k'=15` rate cost) instead.
```
