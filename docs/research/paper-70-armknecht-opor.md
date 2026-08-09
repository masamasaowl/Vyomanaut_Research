# Paper 70 — Outsourcing Proofs of Retrievability

**Authors:** Frederik Armknecht (University of Mannheim), Jens-Matthias Bohli (Hochschule Mannheim / NEC Laboratories Europe), Ghassan Karame (NEC Laboratories Europe), Wenting Li (NEC Laboratories Europe)
**Venue / Year:** IEEE Transactions on Cloud Computing, Vol. 9, No. 1, January–March 2021 (extends ACM CCS 2014)
**Citations:** ~150+ — the OPOR formal model and the Fortress construction
**Topics:** #2 Proof of Storage, #19 Adversarial Defences, #2 Audit Trail
**ADRs produced:** none directly; extends ADR-059's evidence base, directly bears on F-22 and Q68-1
**Findings addressed:** F-01, F-22 (directly), F-32 (evidence)
**Reading list:** Domain A / **R-04** — "publicly verifiable / third-party audit" — accept criterion *"verification without the data owner online"* is this paper's entire subject

---

## Problem Solved

Every construction in Papers 66–68 assumes the verifier is honest. Vyomanaut's verifier is the microservice, and ADR-059 says the quiet part directly: *"The microservice holds a symmetric secret per file and can forge a passing proof, or fabricate a failure. This is F-22 restated and it is not closed here."* F-22 itself, independently raised in the ADR-001–015 interrogation, states the same gap from the provider's side: *"A log held solely by the party it protects is not evidence... The microservice can decline to countersign and the provider has no recourse... Providers must trust the microservice in V2."*

Armknecht, Bohli, Karame & Li name this the **outsourced proof of retrievability (OPOR)** problem: a data owner delegates the audit itself — not just data storage — to a third party, an **auditor**, and now needs security guarantees even when the auditor is dishonest, colludes with the storage provider, or is simply lazy. Every prior POR/PDP model, including Papers 66–68, treats the verifier as trusted by definition. This paper is the first to formalise what happens when it is not, and — critically — it supplies a construction, **Fortress**, built directly on the private Shacham–Waters PRF-based scheme that ADR-059 already adopted. This is not a competing primitive; it is an accountability layer for the one already chosen.

---

## Key Findings

### The formal model: two new security properties

- **Extractability** — the standard POR guarantee (a passing audit implies the file can be extracted), now defined against a threat model where **any subset** of {user, auditor, provider} may be corrupted. A scheme is only **weakly** `ε`-extractable if it needs the user to be honest; Fortress achieves full `ε`-extractability under **any** single honest party.
- **Liability** (`δ`-liable) — the property F-22 is actually asking for: an honest auditor can produce an irrefutable cryptographic proof that it executed the protocol correctly, so that if a file is later found lost, the dispute is decidable rather than a matter of whose word is trusted. A dishonestly-generated log passes this check with probability at most `δ`.

### GetRandomness — depriving the auditor of challenge-selection freedom

The core attack a malicious-but-undetected auditor can mount is choosing *favourable* challenges — either colluding with the provider on which blocks will be asked for, or simply not bothering to check. Armknecht et al. close this by making challenge generation **deterministic given external, unpredictable, publicly-reconstructible randomness**, instantiated via Bitcoin block hashes: `GetRandomness(t)` returns the hash of the first block mined after time `t`, unknowable in advance, verifiable by anyone after the fact. The auditor's actions become a pure function of `(GetRandomness(t), tA)` — any deviation is detectable later by re-running the same deterministic procedure and comparing.

Vyomanaut does not want a blockchain dependency, and does not need Bitcoin specifically — the paper's own requirement is only a source that is **unpredictable before, and publicly reconstructible after**. ADR-015's Transparent Merkle Log, if it gains the gossip mechanism Domain G's R-23 already identifies as missing, is a candidate substrate already inside the project's own architecture. This connects two previously separate reading-list domains (A and G) at exactly the point ADR-059's Q68-1 leaves open.

### Fortress: two parallel PORs, one owner-verifiable and one auditor-verifiable

Fortress does not give the auditor the owner's keys. Instead, at `Store`, **both** the owner and the auditor independently generate their own PSW-style key material — `(k_prf, α_1..α_s)` for the owner, `(k'_prf, α'_1..α'_s)` for the auditor — and the server ends up holding **two** tags per block, `s_i` (owner's) and `s'_i` (auditor's). Every `POR` round runs both challenge-response protocols in parallel against the same server, using randomness derived from `GetRandomness`. The auditor verifies its own `s'_i`-based response immediately, every round. The owner's `s_i`-based responses are logged and can be **batch-verified later**, occasionally, when the owner happens to be online — the paper is explicit this can be *"much less frequent"* than the audit itself.

**The auditor's own tag-generation parameters are proven correct in zero-knowledge at `Store` time**, without revealing `k'_prf` or `α'_j`. The construction commits to them (`g_j = g^{α'_j} mod N`, `h_i = g^{f_{k'_prf}(i)} mod N`, over a modulus `N` of unknown factorisation to the auditor) and proves knowledge of the discrete logs via a non-interactive Schnorr protocol. If the auditor later claims a different `s'_i` than what the committed parameters imply, the two candidate values yield, via Miller's Lemma, a factorisation of `N` — cryptographically infeasible, not merely against policy. **This is a formal answer to exactly the accusation ADR-059 makes against itself**: the auditor cannot silently substitute different key material after the fact.

### Measured cost

Against plain private Shacham–Waters POR (the PSW scheme ADR-059 already adopted) and against the paper's own BLS- and RSA-based generic OPOR transformations:

- Fortress's per-audit POR latency is **~10% above plain PSW**, versus 93% and 85% *improvement* over the BLS- and RSA-based OPOR variants respectively — Fortress is close to free relative to the primitive already chosen, and dramatically cheaper than the pairing-based alternative.
- Fortress's `Store` cost on the auditor is ~10× plain PSW's, driven by ZKP commitment generation — but this is a **one-time cost at upload**, not a recurring audit cost.
- Log-entry size is ~32 KB per audit round (versus ~16 KB for plain PSW) — a 100% blow-up on the **auditor's own log**, not on provider-stored data, since it now holds both PORs' responses and signatures.
- POR verification throughput: Fortress and Fortress-2048 are **~600–1,000× faster** than the RSA- and BLS-based generic OPOR constructions respectively, at the same security level.

### The generic transformation, and why it is not the fit here

Armknecht et al. also give a generic recipe for turning *any* public POR into an OPOR (Section 3), instantiated over both BLS-based and RSA-based Shacham–Waters variants. Both inherit full public-key arithmetic cost per audit — pairings for BLS, modular exponentiation for RSA — which is exactly what R-04's own reject-if criterion in the reading list flags: *"Bilinear pairings at a cost the 5% CPU budget can't carry."* Fortress exists specifically because the authors found the generic transformation too expensive for the private-key case and built a purpose-fit alternative. **Fortress, not the generic BLS/RSA transformation, is the part of this paper relevant to Vyomanaut.**

---

## Substitution at Vyomanaut's parameters

### What adopting Fortress's shape would cost, concretely

Under ADR-059's current design the microservice holds one set of `(k_prf, α_1..α_64)` per file and one 4,096-byte authenticator table per 256 KB chunk (1.5625% overhead). Under a Fortress-shaped adaptation, the **owner** would keep its own keys secret (never handed to the microservice), and the **microservice** would generate and hold a second, independent key set, doubling the authenticator overhead to roughly **3.125% of stored bytes** — still small relative to RS(16,56)'s 3.5× expansion, but not free. The microservice's per-file `Store`-time cost grows to include ZKP commitment generation over a fresh RSA modulus, a one-time cost per file at upload rather than per audit.

The owner does not need to be online for the system to keep functioning day to day — the auditor-side POR runs and gates payment exactly as ADR-059/060 already specify. The owner's `CheckLog` batch-verification of its own parallel PORs is a background reconciliation task, run whenever the owner's client happens to be reachable, which is compatible with "owner offline by design" (ADR-004) precisely because liability is a dispute-resolution property, not a continuous-operation one.

### GetRandomness without a blockchain

Vyomanaut's project constraints rule out a blockchain dependency outright. The paper's actual requirement — unpredictable before, publicly reconstructible after — does not require Bitcoin specifically; it requires *some* append-only, gossip-resistant-to-equivocation source. ADR-015's Transparent Merkle Log is architecturally the right shape, but Domain G's own finding (F-22, restated in that domain's framing, and R-23) is that the log currently has **no gossip protocol**, so an operator could serve different roots to different providers and remain undetected — which would let a colluding microservice defeat `GetRandomness`'s core property the same way it defeats everything else in F-22. Adopting Fortress's accountability model therefore has a real prerequisite: R-23 (split-view detection and log gossip) is not optional infrastructure for this, it is load-bearing.

### A cleaner alternative reading of Q68-1

ADR-059's Q68-1 frames the choice as "accept the random-oracle deviation, or move to BLS to get public verifiability." Fortress supplies a third framing this paper makes newly visible: **keep the private PRF-based scheme (no pairings, no random-oracle-vs-standard-model trade at all, since Fortress's own security proof is stated for the standard model given a working `GetRandomness`), and add liability as a separate, parallel, comparatively cheap mechanism.** This does not eliminate Q68-1's underlying tension — Fortress still needs *some* trusted-but-verifiable randomness source, which is a real dependency — but it reframes the choice from "give up the private scheme's efficiency for accountability" to "keep the private scheme and add accountability at ~10% overhead." The council should see this framing before ruling.

### A quantum-security side note this paper surfaces by contrast

Fortress's core POR remains PRF- and hash-based throughout; only the one-time `Store`-side ZKP commitment uses an RSA modulus. This means the bulk of Fortress's operational cost sits on primitives (HMAC-SHA-256, field arithmetic) that degrade gracefully under Grover's algorithm rather than collapsing outright under Shor's — unlike the BLS pairing alternative Q68-1 also names, which a cryptographically relevant quantum computer breaks completely. This is a minor point relative to F-22, but it strengthens the case for the private-scheme-plus-Fortress path over the BLS path on a second, independent axis. See Paper 72 for the fuller post-quantum picture.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Fortress (Fortress-on-PSW) | Generic BLS/RSA OPOR transformation | ~600–1,000× faster verification; no pairings; requires a second, auditor-owned key set and a one-time ZKP at upload |
| Two parallel PORs (owner-verified + auditor-verified) | One POR with shared keys | Full extractability under any single honest party, not just weak extractability assuming an honest user; doubles authenticator storage |
| Deterministic, externally-anchored challenge generation (`GetRandomness`) | Auditor-chosen challenges | Removes the auditor's ability to cherry-pick or collude on challenge selection; requires an unpredictable, publicly-reconstructible randomness source the project does not yet have |
| Liability via ZKP-backed commitment, checked after the fact | Trusting the auditor's log at face value | Dishonesty becomes cryptographically provable rather than merely suspected — this is F-22's actual request |
| Batch, infrequent owner verification (`CheckLog`) | Owner verifies every audit | Compatible with an offline-by-design owner; defers accountability to dispute time rather than continuous operation |

---

## Breaks in Our Case

- **The auditor is a standalone party the user contracts with, paid to take on liability** ≠ **Vyomanaut's microservice is the coordination layer for the entire network — provider assignment, payment release, repair triggering — not a separately incentivised auditing service**
  → Liability here means "provably followed the protocol," not "financially bonded third party." Vyomanaut can adopt the cryptographic mechanism without adopting the paper's commercial framing (Section 1's SLA-and-remuneration model does not need to be built).

- **`GetRandomness` instantiated via Bitcoin, a mature, already-public, already-gossiped randomness beacon** ≠ **Vyomanaut has no blockchain dependency by design and no working gossip protocol on its own transparency log**
  → Any adoption of this mechanism is gated on Domain G's R-23 landing first. This is a real, not cosmetic, prerequisite — without gossip, the log itself is exactly as equivocable as the auditor Fortress is trying to keep honest.

- **The user is assumed capable of running `CheckLog` and `ProveLog` — non-trivial cryptographic verification — on their own device** ≠ **Vyomanaut's data owner is a consumer with a client application, not a party expected to run cryptographic dispute-resolution tooling**
  → `CheckLog`/`ProveLog` would need to be built into the owner's client app as background, opt-in functionality, likely invoked automatically rather than manually — an engineering scope this paper does not price.

- **Fortress assumes a single auditor per file, contracted once at `Store`** ≠ **Vyomanaut's (3,2,2) gossip-based microservice cluster (ADR-025, ADR-048) has no single fixed "the auditor" identity — responsibility rotates via `ResponsibleReplica()`**
  → The auditor-side key set `(k'_prf, α'_1..α'_s)` would need to be shared across the replica cluster (all replicas can verify), which is a smaller ask than the owner-side keys already being shared in ADR-059's current design, but it is a new requirement this paper does not address directly.

---

## Decisions Influenced

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `PROPOSED — evidence added, no status change`
  Adds this paper to the research source list and to Q68-1's open constraint text: Fortress gives a concrete, costed accountability mechanism for the already-chosen private PRF-based primitive, at ~10% overhead versus the BLS alternative's pairing cost, contingent on a working `GetRandomness`-equivalent (ADR-015 + Domain G's R-23).
  *Because:* F-22 is explicitly named as unclosed by ADR-059 itself; this paper is the first source in the corpus that closes it formally, conditional on infrastructure Vyomanaut does not yet have.

- **No new ADR produced.** Adopting Fortress's shape is a genuine architectural expansion — a second key set, a ZKP step at upload, a dependency on log gossip — and per the project's stated preference this is council territory, not a documentation-only change. Recorded as a sharpened option in ADR-059's Open constraints and as new open questions below.

- **Connects to [ADR-015](../decisions/ADR-015-audit-trail.md) [#2 Audit Trail] and Domain G (R-23).** `GetRandomness`'s requirement (unpredictable, publicly reconstructible, gossip-resistant) is now explicitly a prerequisite for closing F-22 via this path, not only a Domain-G-internal concern.

---

## Disagreements

- **Against reading Fortress as a drop-in fix.** It is tempting to read "Fortress is built on the same PSW scheme ADR-059 already chose" as meaning adoption is nearly free. It is not: it doubles authenticator overhead, adds a one-time ZKP step, and — most materially — depends on a randomness-and-gossip infrastructure Vyomanaut does not currently have. The overhead numbers in this note are the actual cost, not a rounding error.

- **Against the paper's own framing of "weak" extractability as a minor caveat.** Section 3's generic transformation only achieves extractability assuming an honest user; the paper calls this "rather a formal artefact than a real security disadvantage." For Vyomanaut, where the "user" is a consumer with no cryptographic expertise and the party most likely to be a target of social engineering or credential compromise, this caveat deserves more weight than the paper gives it — one more reason Fortress's full extractability (not requiring the generic transformation's weaker guarantee) is the right target if this path is taken at all.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q70-1 and Q70-2 (new). Directly bears on Q68-1, which is updated with this paper's evidence.
