# Paper 72 — Post-Quantum Auditing for Outsourced Storage with Probabilistic Guarantees (PQ-Audit)

**Authors:** Hushuang Zeng, Lina Chen, Zhede Gu, Shanghao Wu, Jiajie Pan (Guangxi Power Grid Co.), Yongguang Yan (Guangdong Power Grid Co.)
**Venue / Year:** Peer-to-Peer Networking and Applications, 19:93 (2026)
**Citations:** newly published (2026)
**Topics:** #2 Proof of Storage, #10 Key Management
**ADRs produced:** none — no near-term architectural fit; recorded as a forward-looking risk
**Findings addressed:** none closed; surfaces a new consideration for Q68-1
**Reading list:** new gap, outside R-01–R-04's stated scope — post-quantum resistance is not asked for by any current Domain A entry

---

## Problem Solved

None of Papers 66–70 consider quantum adversaries. Ateniese's RSA tags, Shacham–Waters' BLS variant, DPDP's RSA-tree option, and Armknecht et al.'s Fortress ZKP all rest, at least partially, on hardness assumptions (factoring, discrete log, pairing-based problems) that Shor's algorithm breaks completely given a sufficiently large fault-tolerant quantum computer. Zeng et al. design **PQ-Audit**, a PDP-style sampling audit built entirely from **hash-based signatures** — specifically a SPHINCS+-style construction combining WOTS+ (a one-time signature over hash chains), FORS (a few-time signature), and a hypertree (a Merkle-tree-of-Merkle-trees) — whose security reduces only to hash-function collision and preimage resistance, believed to degrade gracefully rather than collapse under quantum attack.

This paper is documented because it is current (2026), directly adjacent to Vyomanaut's audit domain, and its own detection-probability theorem (Theorem 1) is the identical Ateniese-style bound `Pr[Reject] ≥ 1 − C(n−t,c)/C(n,c)` already underwriting ADR-060 — useful independent corroboration. It is not documented as a candidate for near-term adoption; the reasons are structural, and stated plainly below.

---

## Key Findings

### The construction

Each outsourced data object `D_i` is committed as `M_i = H(i‖H(D_i))` and signed **once, at setup**, with a stateless hash-based signature over `M_i`: a FORS few-time signature authenticates a message digest, and a hypertree of WOTS+ one-time signatures authenticates the FORS public key up to a single top-level root `PK.root`. The signature (`SIG_i`) is stored alongside the data object on the server. Audits sample a random subset of indices `I` via a symmetric PRF keyed by a private `k_audit` and a fresh nonce, request `(D_i, SIG_i)_{i∈I}` back, and verify each signature independently against `PK` — no interaction with the signer required after setup, and the auditor holds no secret the server can extract or forge against.

### Detection probability — the same theorem, independently derived

Theorem 1 gives `Pr[Accept] ≤ (n−t choose c)/(n choose c) ≤ (1 − t/n)^c`, identical in form to Ateniese's `P_X` bound (Paper 66) and to ADR-060's own derivation. This is now the fourth independent source (Ateniese, Juels–Kaliski, Erway et al., and this paper) reaching the same sampling-detection relationship — strong convergent evidence that ADR-060's formula is not fragile to which primitive family sits underneath it.

### Signature size dominates everything

Six parameter sets are given at three security levels (128/192/256-bit), each with a size-optimised (`s`) and speed-optimised (`f`) variant. Measured signature sizes range from **7,856 bytes** (128s) to **49,856 bytes** (256f) — **two to twelve times** ADR-059's entire per-256-KB-chunk authenticator budget (4,096 bytes) for a **single signed object**, before any aggregation across a sampled set. Signing is the dominant cost by a wide margin over both key generation and verification — the authors' own benchmark shows signing taking roughly an order of magnitude longer than verification at every parameter set, because signing traverses the full hypertree authentication path while verification checks only one.

### No aggregation

This is the decisive property, not a footnote. Each `SIG_i` authenticates exactly one object independently; there is no homomorphic or algebraic combination across multiple objects into a single, smaller proof. The paper's own experiments confirm this: verifying a sampled batch costs the sum of each individual signature's verification, with no batching discount modelled or measured.

---

## Substitution at Vyomanaut's parameters — why this does not fit as specified

### It breaks the one property ADR-060 is built around

ADR-060's entire sampling design rests on Shacham–Waters' constant-size aggregate response: *"Response size is constant in the number of chunks audited, which is what makes ADR-060's one-receipt-per-`(provider, file, day)` aggregation possible and is the mechanism behind F-02's fix."* PQ-Audit's per-object hash-based signatures do not aggregate. Applying PQ-Audit at Vyomanaut's actual sampling rate — 2,867 chunks × 256 blocks sampled per provider per day under ADR-060, or even at chunk-level granularity, 2,867 objects per provider per day — would require transmitting and verifying 2,867 **separate** signatures at 7.8–49.9 KB each: **22–143 MB per provider per day**, against the current design's constant 1,040 bytes regardless of sample size. This is not a tuning gap; it is the reason ADR-059/060's whole design exists, undone.

### Where it could fit, if it ever needed to

Signing once and verifying repeatedly is actually a reasonable match to Vyomanaut's usage pattern — tag at upload, audit continuously — which is the same shape DPDP's per-file root or a Merkle-tree-per-chunk option already has. A PQ-Audit-style construction applied at a much **coarser granularity** — one hypertree signature per file or per stripe, not per chunk — would shrink the per-object count by roughly three orders of magnitude and could be evaluated on its own terms. This paper does not propose that adaptation; it is only a implication of reading the construction against Vyomanaut's numbers, recorded here for a future revisit rather than pursued further now.

### The current choice is already more PQ-resilient than the alternative it was weighed against

ADR-059 chose the PRF-based Shacham–Waters **private** scheme over the BLS **public** variant. HMAC-SHA-256 and modular field arithmetic (the private scheme's actual primitives) degrade under Grover's algorithm — roughly halving the effective security parameter, addressed by widening `p`, not by a structural redesign. BLS pairings, by contrast, are fully broken by Shor's algorithm the moment a sufficiently large quantum computer exists. Reading PQ-Audit's motivation against ADR-059's existing choice surfaces a fact that was not previously stated in any Vyomanaut document: **the primitive already selected carries meaningfully less long-term quantum risk than the public-verification alternative it was weighed against in Q68-1.** This does not argue for adopting PQ-Audit; it argues that abandoning the private scheme for BLS (one of Q68-1's two options) would trade a bandwidth problem for a much larger, harder-to-reverse one.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Hash-based signatures (SPHINCS+ family) | RSA/discrete-log/pairing-based tags | Security reduces only to hash-function properties, believed post-quantum-resistant; signatures are 2–12× a comparable chunk's entire current authenticator budget, with no aggregation |
| Per-object independent signing | An aggregatable per-file or per-stripe structure | Simple, stateless verification per object; verification cost and bandwidth scale linearly with sampled objects, with no batching discount |
| Size-optimised vs speed-optimised parameter sets | A single fixed parameter choice | Lets a deployment trade minutes of signing time for roughly 2× smaller signatures, or vice versa; neither point is close to Vyomanaut's per-chunk overhead budget |

---

## Breaks in Our Case

- **Signatures are 7.8–49.9 KB per object, with no aggregation across sampled objects** ≠ **ADR-059's entire chunk overhead budget is 4,096 bytes, and ADR-060's audit response is 1,040 bytes constant regardless of sample size**
  → Direct adoption at chunk granularity is a bandwidth and storage regression by one to two orders of magnitude on the property Vyomanaut's audit design was specifically built to hold constant. Not viable as specified.

- **The paper assumes no existing homomorphic-tag infrastructure to preserve compatibility with** ≠ **ADR-059 already commits to a PRF-based aggregate scheme, and a hash-based signature scheme cannot be layered underneath or combined with it — the two are structurally incompatible primitive families**
  → Adopting PQ-Audit would mean replacing ADR-059 wholesale, not extending it. This is a large, currently unmotivated redesign, since no post-quantum NFR or compliance requirement exists anywhere in the current requirements set.

- **The paper's threat model is a generic quantum-capable adversary against the storage server** ≠ **Vyomanaut's Ed25519 provider-signing scheme (a project-wide architectural constant, per house context) is itself not post-quantum-secure**
  → If a post-quantum requirement ever does arrive, it would not be scoped to the audit primitive alone; Ed25519 signing throughout the system would need the same reconsideration. This paper only covers one corner of a much larger migration that is not currently in scope anywhere.

---

## Decisions Influenced

- **No ADR produced.** No current requirement calls for post-quantum resistance, and direct adoption breaks ADR-059/060's core design property. Recorded as a forward-looking risk note, not a decision.

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `EVIDENCE ADDED — no status change`
  Adds a previously unstated point to Q68-1: the private PRF-based scheme already chosen is meaningfully more quantum-resilient than the BLS alternative it was weighed against, independent of the random-oracle-versus-standard-model trade-off that question is otherwise about.
  *Because:* this changes the shape of Q68-1's trade-off table, even though it does not resolve the question.

---

## Disagreements

- **Against treating this as urgent.** Nothing in Vyomanaut's requirements, MVP, or NFR set currently calls for post-quantum security, and Ed25519 is a project-wide constant used well beyond the audit path. Prioritising a full audit-primitive redesign now, ahead of a stated requirement, would be solving a problem the project has not yet decided to have. Recorded for awareness, explicitly not recommended for near-term work.

- **Against reading the "one-time signature, multiple verifications" framing as a natural fit without qualification.** The paper's own conclusion states this shape suits its scheme well, and in isolation the sign-once/verify-often pattern does match Vyomanaut's usage. The paper's own numbers show this framing does not survive contact with Vyomanaut's actual sampling volume, since verification cost scales with objects sampled, not with objects that exist — a distinction the paper does not need to make for its own evaluation (which does not model continuous high-frequency sampling at Vyomanaut's scale) but which is decisive here.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q72-1 (new). Not blocking; recorded as a V3/future-facing item.
