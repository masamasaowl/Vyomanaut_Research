# Paper 77 — Secret Sharing Made Short

**Authors:** Hugo Krawczyk (IBM T.J. Watson Research Center)
**Venue / Year:** CRYPTO '93, LNCS 773, pp. 136–146, 1994
**Track:** LTS
**Reading list:** Domain P / R-29 — Band 1 — accept criterion *"reconstruction distributed so no
participant sees plaintext, at stated round/latency cost"*
**ADRs produced:** [ADR-022 Addendum B](../decisions/ADR-022-addendum-b-ramp-secret-sharing-rejected.md) (new)
**Findings raised:** none new **Questions closed:** none · **Questions raised:** none
**Triage score:** 6/10 (parameter reach 2 · trust model 1 · evidence 1 · actionability 2 · corpus delta 0)
— drafted despite a mine-the-introduction score because its abstract's lower bound is the whole
reason it is cited in `reading-list-LTS.md`'s must-read table; the construction itself turns out to
be closely related to, not a new option beyond, AONT-RS.

---

## Provenance

Read: the full 11-page CRYPTO '93 proceedings scan (OCR quality is imperfect in places —
`w — h i c h` style spacing artifacts — but the mathematical content is legible throughout). No
proofs were skipped; the paper is short enough to read completely.

## Problem Solved

Perfect (information-theoretic) secret sharing has a proven floor: **a share must be at least as
large as the secret** `[P §Abstract]`. For a large secret — a file, not a key — Shamir's scheme at
this floor means every one of `n` shareholders stores a **full copy** of the secret's size. Krawczyk
asks whether *computational* secrecy (against resource-bounded adversaries) can beat this floor, and
answers yes: his scheme, **SSMS** (Secret Sharing Made Short), gets shares down to `S/m` plus a
security-parameter-sized constant, for an `m`-threshold scheme protecting a secret of size `S`
`[P §Abstract]`.

## Key Findings

### The construction is encrypt-then-disperse-then-share-the-key

`[P §1]` SSMS combines three primitives: (1) encrypt the secret `S` with a private-key cipher under a
random key `K`; (2) disperse the ciphertext with an information dispersal algorithm (Rabin's IDA,
`(m,n)`-threshold, cost `n·|S|/m`); (3) share the short key `K` itself with a **perfect** (Shamir)
secret-sharing scheme, cost `n·|K|`, where `|K|` is a security parameter independent of `|S|`.
Reconstruction needs `m` of the `n` (ciphertext, key-share) pairs.

### This is provably secure given only a one-way function

`[P §1]` The scheme requires nothing stronger than the existence of a secure private-key encryption
system — *"in particular, it just requires the existence of a one-way function"* `[P §1]`.

### It has no proactive-refresh mechanism

`grep` of the full text for `proactive`, `refresh`, and `share renewal` returns nothing. This paper
predates proactive secret sharing (Herzberg et al., 1995) by two years and does not address it. Any
Domain P claim that SSMS-style shares can be periodically re-randomized against long-term
accumulation is **not supported by this paper** — that mechanism, if it exists in the corpus at all,
belongs to Paper 78's neighbour, the (uploaded but not yet obtained) Eldefrawy et al. PSS↔RC paper.

## Substitution at Vyomanaut's Parameters

`[DERIVED]` **The fundamental lower bound, at our scale.** A perfect (IT) Shamir scheme storing the
segment directly across `n = 56` nodes requires every share to be at least the size of the secret
`[P §Abstract]`. Total system storage for one segment: `n × (segment size) = 56×`. This is the exact
cost of full IT security with no computational shortcut — **56× expansion**, working: shares ≥
secret size (proven lower bound) × 56 nodes = 56× total.

`[DERIVED]` **SSMS's own cost, at our `(n,m)=(56,16)`:** ciphertext dispersal at Rabin's rate gives
`n·|S|/m = 56/16 = 3.5×` — **identical to RS(16,56)'s existing expansion** — plus `n·|K|` for the
key shares. At `|K| = 32` bytes (a 256-bit key) and a 256 KiB shard, the key-share term adds
`56×32 B = 1,792 B` per segment against a `~14 MB` segment (`n × 256 KiB`), i.e. **0.0125% overhead**,
negligible.

### The comparison this substitution actually supports

`[INFERRED]` SSMS is not a *third option* Domain P should weigh against AONT-RS. It is AONT-RS's
direct architectural ancestor, and Paper 16 says so explicitly in its own related work: *"In 1993,
Krawczyk proposed a blending of Rabin and Shamir... Our dispersal algorithm... also enriches Rabin's
IDA with security. Unlike SSMS, it does so without secret sharing"* `[P — cited from Paper 16, not
this paper]`. AONT-RS **is** SSMS with the last inefficiency removed: instead of Shamir-sharing `K`
separately (the `n·|K|` term above), AONT-RS embeds `K` in the package itself via `K ⊕ h`, eliminating
even that negligible overhead. **Vyomanaut is already running the improved version of this scheme.**

## What This Paper Rules Out

- **Rules out full information-theoretic secret sharing as an economically viable path for Domain P.**
  The `56×` figure above is not a design choice being weighed against `3.5×` — it is a **proven lower
  bound** on any scheme with SSMS's perfect-secrecy guarantee applied directly to the segment. No
  amount of clever engineering reduces it while keeping IT security intact; the paper's abstract
  states the bound is *"clearly optimal"* if the secret must be recovered from `m` shares.
- **Rules out treating this paper as a candidate replacement for AONT-RS.** It is dominated by what
  Vyomanaut already runs, on the paper's own logic (Paper 16's related-work comparison).
- **Adjacent-not-this, confirmed:** the paper is filed in R-29 under "threshold cryptography applied
  to repair," but it says nothing about *repair* at all — it is a dispersal-time construction, like
  AONT-RS, not a repair-time protocol. It does not belong next to Huang & Bruck (Paper 78) in
  function, only in keyword.

## Trade-offs

| Chosen (already, via AONT-RS) | Over (this paper's own construction) | Consequence |
|---|---|---|
| Embedded key (`K ⊕ h` in the AONT package) | Shamir-shared key (SSMS) | Removes the `n·\|K\|` term entirely — a negligible saving in bytes, but also removes a second cryptographic primitive (Shamir) and its own key-management surface |
| Computational security via `K` enumeration | Perfect security via share-size floor | `3.5×` vs `56×` — the entire reason Domain P is not proposing IT security as the answer to F-69 |

## Breaks in Our Case

- **The paper's `(m,n)`-threshold model assumes a static file, no repair events** ≠ **Vyomanaut's
  segments are repaired repeatedly over a multi-year lifetime**
  → **Fatal for direct application, Costed as an ancestor-lineage lesson.** SSMS says nothing about
    what happens when a share is regenerated after loss — that is precisely the gap Papers 76 and 78
    exist to fill. This paper's contribution to Domain P is the **lower-bound argument**, not a
    repair-time construction.

## Decisions Influenced

- **[ADR-022 Addendum B](../decisions/ADR-022-addendum-a-cap-is-not-confidentiality.md) — new,
  drafted below.** `ACCEPTED` (addendum status). Formally closes the option "replace AONT-RS with a
  ramp/IT secret-sharing scheme to buy repair-time blindness" on cost grounds: `56×` vs `3.5×`. This
  is a negative result and, per `reading-list-LTS.md` §3.6, a recorded deliverable in its own right.

## Falsifiers

- **A parameter change.** If Vyomanaut's threat model is ever revised to require security against a
  computationally unbounded adversary (post-quantum key-recovery concerns are the likeliest driver,
  tracked at Q72-1), the `3.5×` AONT-RS/SSMS-lineage answer stops being available and the `56×` floor
  becomes the honest cost of the requirement — this note's comparison inverts.

## Disagreements

None — this paper predates and is consistent with everything else in the corpus that cites it.

## Corpus Delta

Confirms rather than extends the corpus: formalizes, with a citable proof, the reasoning implicit in
`reading-list-LTS.md`'s framing that "computational tricks beat IT bounds" for Domain P. No existing
note is corrected. The one substantive addition is the explicit `56×` lower-bound number, which did
not exist anywhere in the corpus as a derived figure before this note.

## Open Questions

None raised. No manual step required for this paper beyond filing the ADR addendum below.
