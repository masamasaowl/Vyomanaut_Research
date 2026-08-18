# ADR-022 Addendum B — Ramp / IT Secret-Sharing Considered and Rejected for Repair Blindness

**Status:** Accepted
**Track:** LTS
**Topic:** #3 Confidentiality / #15 Encryption-Erasure Interaction
**Amends:** ADR-022 (closes an alternative the original ADR did not consider; the AONT-RS decision
itself is unaffected and further confirmed)
**Research source:** Domain P literature session, August 2026 — Paper 77 (Krawczyk, Secret Sharing
Made Short)

---

## The question this addendum closes

Domain P's search for a repair-blind confidentiality mechanism raised an implicit alternative: instead
of adjusting AONT-RS or adding a secure-repair protocol on top of it, **replace the encoding pipeline
with a true information-theoretic (perfect) secret-sharing scheme**, which by construction reveals
nothing to any party holding fewer than the reconstruction threshold — repair included, with no
protocol work needed. This addendum records why that alternative is rejected, as a permanent,
citable answer rather than a question that recurs each time confidentiality is revisited.

## The finding

Krawczyk's paper states the governing lower bound in its own abstract: *"shares must be of length at
least as the secret itself"* for perfect secrecy `[Paper 77]`. Applied directly to Vyomanaut's
`n = 56`: storing a segment as `n` full-size perfect-secrecy shares costs

```
n × (segment size) = 56×
```

against AONT-RS's current `3.5×` (`n/k = 56/16`). **This is not a design choice weighed against a
comparable alternative — it is a proven floor.** No implementation cleverness reduces it while
keeping the perfect-secrecy guarantee intact.

Krawczyk's own paper is itself a case study in why nobody pays this price: his construction (SSMS)
achieves *computational* secrecy at `n·|S|/m + n·|K|` — essentially AONT-RS's own cost structure,
predating it by 14 years. AONT-RS (Paper 16, ADR-022's basis) is SSMS with the residual `n·|K|` term
removed by embedding the key in the package itself. **Vyomanaut is already running the improved
descendant of the scheme that first demonstrated computational security beats the perfect-secrecy
floor at acceptable cost.**

## Decision

1. **Replacing AONT-RS with a perfect (information-theoretic) secret-sharing scheme is rejected**, on
   cost grounds: `56×` vs `3.5×`, a proven floor rather than an estimate.
2. **This closes the premise Domain P's protocol-layer candidates (Huang & Bruck, Paper 78; the
   MSR/LRC/determinant constructions, Papers 76/80/81/82) all depend on** — namely, that AONT-RS (or
   a code carrying a small, tunable secrecy margin `z`) remains the right foundation, and the
   confidentiality-under-repair problem is a *repair-protocol* problem layered on top of it, not an
   *encoding-scheme* problem requiring a wholesale replacement.
3. **No change to ADR-022's decision.** AONT-RS as the encoding pipeline stands, further confirmed
   rather than revised.

## Consequences

**Positive.** A negative result — "do not pursue this" — is recorded once, permanently, with a
citable proof, rather than being re-litigated informally each time Domain P or a future confidentiality
review revisits the question. Per `reading-list-LTS.md` §3.6, a recorded null result is a deliverable
in its own right.

**Negative.** None.

**Open constraints:** if Vyomanaut's threat model is ever revised to require security against a
computationally unbounded adversary (the likeliest driver is a post-quantum key-recovery concern,
tracked at Q72-1), this addendum's cost comparison inverts and the `56×` floor becomes the honest
price of that requirement, not a rejected alternative.

## References

- ADR-022 — the encoding-pipeline decision this addendum confirms
- Paper 16 (Resch & Plank, AONT-RS) — the scheme Vyomanaut runs, and SSMS's stated architectural
  descendant per Paper 16's own related-work section
- Paper 77 (Krawczyk, Secret Sharing Made Short) — the source of the `56×` lower bound
- ADR-079 (new) — Domain P's Menu, which this addendum's closure keeps scoped to protocol-layer and
  small-margin codec options rather than a wholesale encoding replacement
