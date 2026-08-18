# ADR-079 — Domain P Menu: Confidentiality-Preserving Repair, Priced

**Status:** Proposed
**Track:** LTS
**Topic:** #3 Confidentiality / #4 Repair Protocol
**Supersedes:** —
**Superseded by:** — *(to be superseded by whichever council decision selects among the options
below; this ADR is a menu, not a selection)*
**Research source:** Domain P literature session, August 2026 — Papers 76, 77, 78, 79, 80, 81, 82;
Paper 16 LTS addendum

---

## Context

`ADR-076` established that under RS(16,56) + AONT-RS as currently deployed, every candidate repairer
is disqualified from confidentiality (F-69) — a party either assembles the plaintext-adjacent AONT
package or cannot carry durability. `ADR-076 Addendum A` narrows this: the disjunction ADR-076 stated
("the code family changes, or a party is trusted") is missing a third branch — a protocol-only or
small-margin codec adjustment that trusts no party and does not replace RS(16,56) wholesale. Domain
P's Band 1 literature reading exists to find and price that branch, and this ADR records what it
found.

**What this ADR is not:** a selection. `reading-list-LTS.md`'s own framing requires a council for a
"genuine tradeoff between defensible positions with no dominant answer," and that is exactly the
shape of what follows — storage cost, durability cost, `r0`-gate compatibility, and protocol
complexity trade against each other with no option dominating on every axis. This ADR's job is to
make that trade-off citable and complete enough for a council to convene against, not to pre-empt the
council.

## The menu

All figures are storage expansion for blindness against a single elected repairer (`ℓ1=0, ℓ2=1`),
the minimal target this domain has scoped to (F-34/ASN-scale blindness was removed from Domain P's
header — see `ADR-022 Addendum A` — as unaffordable at any construction in this menu; see Paper 76's
note for the `82.5×` figure at `ℓ2=15`, one below the wall).

| Option | Source | Expansion at `ℓ2=1` | Repair degree `d` | `r0`-gate (`d≤24`) compatible? | Cost centre |
|---|---|---|---|---|---|
| **P1 — Plain secure MSR** | Papers 76 + 80 | **3.73–3.83×** (cross-validated, two methods) | `n−1 = 55` | **No** — requires full pool | `r0` gate must be raised, or achievability re-derived at `d<n−1` (unproved) |
| **P2 — Secure LRC, `r=2`** | Paper 81 | **4.00×** | `r = 2` | **Yes** | Durability: `dmin=34`, tolerates 33 losses (`−7` vs RS) |
| **P3 — Secure LRC, `r=4`** | Paper 81 | **4.67×** | `r = 4` | **Yes** | Durability: `dmin=38`, tolerates 37 (`−3`) |
| **P4 — Secure LRC, `r=8`** | Paper 81 | **7.00×** | `r = 8` | **Yes** | Durability: `dmin=40`, tolerates 39 (`−1`, matches RS almost exactly) — but worst secrecy cost in the menu |
| **P5 — Secure determinant code, `m=16`** | Paper 82 | **3.73×** | `d = k = 16` | **Yes** | Fixed repair degree, no tuning; near-identical cost to P1 |
| **P6 — Secure determinant code, `m=8`** | Paper 82 | **3.97×** | `d = k = 16` | **Yes** | 50% repair bandwidth vs a naive full re-fetch; a genuine bandwidth/storage dial P1–P4 don't offer |
| **P7 — Two-round dealer-free protocol** | Paper 78 | applies **on top of** whichever code above is chosen; adds no order-of-magnitude cost (`~1.02×` of the code's own bandwidth) | — | — | Removes the *trust* requirement from whichever code is chosen; does not by itself reduce storage |

**Ruled out, not in the menu:** full information-theoretic secret sharing (`56×`, `ADR-022 Addendum
B`); Guruswami–Wootters / Dau et al. sub-symbol repair (Paper 79 — pure bandwidth, no confidentiality
content, reclassified out of this menu entirely); ASN-scale (`ℓ2≈11`) blindness at any construction
here (unaffordable, `ADR-022 Addendum A`).

## What the council still needs, before it can choose

1. **Q79-2** — does Paper 78's protocol (P7) preserve AONT-RS's *computational* security, or only a
   bare information-theoretic `z≥1` code's? Unresolved; determines whether P7 layers onto Vyomanaut's
   actual pipeline or only onto a hypothetical bare-RS-with-padding variant.
2. **Q79-1** — can a single batched lazy-repair cycle (up to 32 shards under the `r0` gate) expose
   `ℓ2` up to 32 to one party in one event, exceeding every option's `ℓ2<16` design wall outright
   rather than accumulating toward it? This is the single largest unresolved risk in the menu and
   applies to **every** option above, not just P1.
3. **Repairer-election distribution** (a code-reading task, not a literature one, per `ADR-076`'s own
   "Open constraints" — Q76-1) — whether repair execution is genuinely one-shard-per-election or
   effectively centralized per batch determines whether Q79-1's worst case is the typical case.
4. **P1's `d<n−1` achievability gap** — Papers 76/80 prove their construction only at the full
   `d=n−1=55` pool; whether it (or an equivalent) can be re-derived at `d=24` under the `r0` gate is
   open and, if it cannot, removes P1 from contention entirely regardless of its attractive cost.

## Decision

1. **The menu above is ratified as the complete, priced Domain P candidate set** for single-repairer
   blindness, pending the four items above.
2. **No option is selected.** This is deferred to a design council, to be convened once items 1–4
   above are resolved or explicitly accepted as residual risk.
3. **Domain P's scope is confirmed as closed for the `ℓ2=1` target.** Further literature search
   within this scope is not warranted — two independent constructions (P1) cross-validate to within
   2.5%, and P2–P6 are drawn from the two remaining relevant papers in the corpus. Effort should move
   to items 1–4, which are code-reading and proof-obligation tasks, not reading-list tasks.
4. **This ADR's status remains Proposed until the council convenes and either selects an option
   (superseding this ADR with the selection) or defers, in which case this ADR is re-affirmed as the
   standing reference menu.**

## Consequences

**Positive.** A council convened on this question now has a citable, cross-validated menu rather than
an open-ended search space. The two independent `~3.8×` results (P1) and the internal consistency of
P5/P6 with P1 at their MSR-equivalent mode give real confidence that the *achievable* cost of
single-repairer blindness, ignoring the `r0`-gate and batching questions, is well-characterized:
roughly 4–7% to 33% storage overhead depending on which durability/bandwidth trade-off is preferred,
not the order-of-magnitude costs earlier informal reasoning worried about.

**Negative.** Every option in the menu is unresolved against the `r0`-gate batching question
(item 2), which could invalidate the entire menu's premise if a single batched cycle routinely
exceeds every option's design wall. This ADR does not resolve that; it makes clear that resolving it
is the actual gating question, not a further literature search.

**Open constraints:** all four items in "What the council still needs" stand as open constraints on
this ADR's own completeness, not merely on the eventual selection.

## References

- ADR-076, ADR-076 Addendum A — the structural problem this menu answers, and the corrected framing
- ADR-022 Addendum A — removes ASN-scale blindness from this menu's scope
- ADR-022 Addendum B — removes full IT secret-sharing from this menu's scope
- Paper 16 LTS addendum — the AONT-RS threat-model boundary this menu's P7 option must cross
- Papers 76, 78, 80 — P1 and P7
- Paper 81 — P2, P3, P4
- Paper 82 — P5, P6
- Paper 79 — explicitly excluded, with reasoning, from this menu
