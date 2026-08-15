# Research Topic List v3.1 — Council Addendum

**Applies to** `reading-list-v3.md`. **Does not replace it** — v3's domain framings, search strings,
must-read tables and substitution tests stand except where amended below. Addendum rather than
rewrite, per the project's standing preference where the change is a *ruling on existing content*
rather than a new structure.

**Source:** the five-session Design Council of August 2026
(`design-council-2026-08-lts-blockers.md`), which cleared the seven blocking items in v3 §7.2.

**Track:** LTS throughout.

---

## §1 — Domains closed

Remove these from the active list. Each is closed by a ruling, not by exhaustion.

### Domain C — Wide stripes and repair I/O → **CLOSED except two topics**

Council 2 settled Q26-4 from the code and Q62-1 followed. The `≤ 24` lazy-repair gate is unbuilt,
so single-fragment repair is currently 100% of events — but the ruling is to **build the gate**, not
bless the behaviour. Once `r0 = 8` is enforced as ratified, every repair event reconstructs 32
fragments and the single-block optimisation literature delivers 0%, exactly as ADR-004's own table
states.

| Topic | Status |
| --- | --- |
| **R-09** — wide-stripe repair and combined locality | **CLOSED.** Worth zero under `r0 = 8` |
| **R-10** — piggybacking code framework | **CLOSED.** Same |
| **Papers 62 (LESS) and 64 (Hitchhiker)** | **NOT DRAFTED.** Removed from Band 0's drafting queue |
| **Q62-2** — MDS-feasible LESS coefficients at `n−k = 40` | **CLOSED UNSTARTED.** ADR-004 already warned the search is expensive and worthless if single-fragment repair never happens |
| **Q64-1** — Hitchhiker mandatory-helper interaction | **CLOSED.** Conditional on a selection that will not happen |
| **R-11** — small-object erasure coding | **SURVIVES.** Independent of repair policy; the 4 MB minimum efficiently-coded object stands regardless |
| **R-64** — redundancy adaptation from observed reliability | **SURVIVES.** Reliability-driven, not single-block-repair-driven |

**ADR-026 should close as *no V3 repair-bandwidth optimisation*.** That retires the council item v3
§7.2 listed as open, and removes a milestone from `build_part4.md`'s map.

### R-49 — verifier-side audit cost → **CLOSED by arithmetic**

Q68-2 resolved without literature. Substituting ADR-060's own parameters: 286,720 chunks over 1,000
files → ~287 chunks/file → 1% sampling → ~2.87 chunks per file-audit → × 256 blocks → ~734 blocks
per file → × 1,000 files = **733,952 PRF evaluations per provider per day**. At 1,000 providers:
7.34 × 10⁸/day ÷ 86,400 = **~8,494 HMAC-SHA-256/s**, a fraction of one core. At ADR-002's
100,000-provider horizon, ~850 k/s — a few cores.

Q68-2 closes. **Note the second-order consequence recorded by the council:** this also removes the
principal cost objection to the BLS variant, so if Q68-1's Fortress path fails, BLS must be
**re-priced**, not rejected on inherited grounds.

### R-60 — deduplication side channels → **CLOSED by construction**

AONT-RS uses a fresh random key per segment, so identical plaintexts produce different ciphertexts
and therefore different `chunk_id`s. The convergent-encryption attack family does not apply.
**One verification is required before this closes formally** (Council 5, recommendation 7): confirm
the client generates a fresh AONT key per segment *per upload*, including on re-upload of an
identical file.

### Domain H — shrinks from a domain to a research note

R-60 closes; **R-59 survives scoped to the squatting case only** — adversarial submission into a
content-addressed namespace, where an attacker submits a `chunk_id` it knows another owner will
produce. Output is a research note, not a `paper-NN`. The must-read list stays useful for the note:
Harnik/Pinkas/Shulman-Peleg and Bellare/Keelveedhi/Ristenpart, read for threat model rather than
construction.

### R-63 — time-weighted attribution → **CONDITIONAL ON A COUNT, do not search**

Its trigger was ADR-061's mid-period repair-handover problem. Under Council 2's ruling repair
becomes rare once the `r0` gate exists, so handovers become rarer still. **Count the handover
fraction first.** If it is under ~1%, R-63 closes unstarted and F-LTS-04 resolves as "flat split,
documented."

---

## §2 — Domains re-ranked

### Domain P — Confidentiality-preserving repair → **now the top priority, above Domain A′**

Council 2's Systems Theorist produced the argument that changes this domain's status. There are
exactly three candidate repairers and each is disqualified by a property of the current code family:

| Repairer | Disqualifying property |
| --- | --- |
| A **provider** | must gather `k = 16` shards → obtains the plaintext |
| The **operator** | obtains the plaintext **and** breaks ADR-019's stated trust model **and** carries all repair egress on one host |
| The **owner** | offline by design (ADR-004); cannot carry durability |

**There is no assignment of the repair role under RS + AONT-RS in which nobody obtains a decodable
package.** F-69 is therefore not a defect to be fixed by relocating the role — it is a proof that
the code family must change or a party must be explicitly trusted with plaintext.

Consequences for the list:

- R-27, R-28, R-29 are unchanged in content and **promoted to first position in Band 1**.
- **R-28 (repair without full reconstruction) is now load-bearing rather than optional** — it is the
  only topic that attacks the impossibility directly.
- Add to every topic in this domain an explicit new **substitution test:** the mechanism must be
  evaluated against a repair event that is **rare** (post-`r0`-gate), not constant. Rarity does not
  make disclosure acceptable, but it changes what a partial mitigation is worth, and no v3 topic
  said so.
- **Recommend swapping Confidentiality ahead of Repair & Erasure Optimisation** in
  `build_part4.md`'s milestone map — the latter largely closes anyway (§1).

### Domain A′ — R-47 becomes the gating read; R-49 leaves

Council 1 declined to rule on Q68-3 for cause: ADR-059 rejected microservice-side tagging because
it *"contradicts ADR-021's pure-P2P repair model,"* and F-LTS-08 shows pure-P2P repair does not
exist. Council 2 then restored provider-side repair on the LTS track, which puts the microservice
option back off the table.

Q68-3 therefore reduces to a two-way choice — give the repairing provider the authenticator keys
(it can then forge proofs for the position it just wrote), or leave reconstructed shards unauditable
until the owner returns — and **neither is acceptable**.

| Topic | Status |
| --- | --- |
| **R-47** — tag generation for reconstructed shards without the owner | **PROMOTED to gating read for the Proof of Storage milestone.** Chen & Curtmola (JCS 2017) and Chen/Ammula/Curtmola (CODASPY 2015) are already in Band 0 and need no search |
| **R-48** — detection composition across a 56-prover stripe | Unchanged. Still open (Q67-1) |
| **R-49** — verifier-side cost | **REMOVED** (§1) |

### Domain G — R-50 promoted to hard dependency

Council 1's deciding argument was the Adversary's **seed-grinding attack**: ADR-060 has the
microservice *"draw a fresh 32-byte challenge seed"* and nowhere states that the seed must be
unpredictable *to the verifier*. A verifier free to choose seeds can grind for one whose derived
1% sample lands only on chunks a colluding provider retained — defeating a 94.3%-detection scheme
with no cryptographic break at all.

This reverses Q68-1's framing. The beacon is not Fortress's *price*; it is the *fix*, and Fortress
arrives with it. **Q70-1 resolves: yes, ahead of Domain G's other priorities.**

- **R-50 is a hard dependency of the Proof of Storage milestone**, not a Band 1 background item.
- **Price consumption before construction.** Consuming an external beacon (`drand` / League of
  Entropy) may reduce R-50 to an integration question. Establish that before any search.
- **New substitution test, from peer review:** ADR-060 draws one seed per `(provider, file, day)` —
  at 10⁶ files that is 10⁶ seeds/day, which no beacon emits directly. The design must be *one beacon
  value per epoch, expanded locally*, and any candidate must survive that expansion without losing
  unpredictability. Show the expansion in the triage note.
- R-23, R-24, R-25 unchanged.

### Domain K — R-17 moves in front of R-30 and R-31

Council 4 sided with the Systems Theorist: **a placement cap cannot repair a collusion threshold**,
because a cap constrains honest placement and the threat is cooperation after placement. Cap-tuning
raises the attacker's price; it does not restore the property.

The Scale Advocate's substitution nonetheless bounds the mitigation: to make two-ASN collusion
insufficient you need `2 × floor(56f) < 16` → `floor(56f) ≤ 7` → `f < 0.143`, requiring **≥ 8
genuinely independent ASNs** to place 56 shards. Peer review sharpened it further — at `f = 0.143`,
**three** colluding ASNs still reach 21 ≥ 16, so cap-tuning buys exactly one coalition size.

- **R-17 is promoted from Band 2 (Domain E) to Band 1 (Domain K)** and re-scoped: it is the
  feasibility gate on whether cap-tuning exists as a mitigation at all. If Indian residential
  broadband cannot span 8 independent ASNs for a consumer pool, cap-tuning is off the table and
  **R-30 is the only remaining instrument**.
- **R-17 absorbs F-LTS-10:** real ASN detection is unimplemented (registrations supply `demo_asn`),
  so the cap is currently enforced against provider-supplied data. And peer review's extension —
  under CGNAT a carrier ASN may cover millions of subscribers, so ASN diversity may overstate
  failure-domain **and** collusion-domain diversity at once. Both fold into R-17's scope.
- **F-34 is re-ranked below F-69.** The operator's by-design reconstruction is the primary
  confidentiality exposure; two-ASN collusion is secondary. Same fix serves both.
- R-30 (Krawczyk first), R-31 unchanged in content, sequenced after R-17.

### Domain N — urgency drops, scope returns to confidentiality

F-LTS-01 closes on Council 5's zero-cost fix (adding `segment_id` + `shard_index` to the
capability-token signing input). Authorisation integrity no longer depends on AONT key freshness.

- **R-39** (RNG failure in the wild) — urgency **reduced**. It remains a real confidentiality input
  but is no longer upstream of the upload authorisation path.
- **R-38** (extended-nonce / misuse-resistant AEAD) — unchanged.
- One verification carries over from §1: confirm fresh AONT key per segment per upload.

---

## §3 — New blocking items entering the list

Five findings from the council's verification pass. Four are prescriptive; one is a milestone.

| # | Item | Type | Where it goes |
| --- | --- | --- | --- |
| **F-LTS-07** | Lazy repair at `r0 = 8` is unimplemented; the system performs eager repair | **LTS milestone.** Needs a per-segment live shard counter, a threshold scan loop, and a fix to `findMissingShardIndex` (derives "missing" from a caller-supplied holder list, unusable by a scanner) | ADR-004 addendum; precedes all Domain P work |
| **F-LTS-08** | `cmd/microservice` performs reconstruction, decoding the AONT package | **Milestone.** Move the executor provider-side; new "elected repairer" protocol; capability tokens for cross-provider writes (nearly free — the token already binds `chunk_id` + `provider_id`) | ADR-004 / ADR-021 addendum |
| **F-LTS-09** | ADR-012's headline contradicted by the implemented payment path | **Prescriptive.** Amend ADR-012: distinguish per-GB **ingress** (rejected on Storj evidence) from per-GB **stored-per-period** (adopted) | One ADR amendment |
| **F-LTS-10** | ASN cap enforced against a self-declared field | **Folded into R-17** | Domain K |
| **F-LTS-11** | ADR-059's 4,096 B of per-chunk authenticators do not fit IC §4.1's fixed 262,252-byte Frame 1, and their relationship to `chunk_id = SHA-256(chunk_data)` is unspecified | **Prescriptive, blocking implementation.** Resolve before any Proof of Storage session | ADR-059 amendment |

Two more from Council 1, not numbered above because they are amendments rather than findings:

- **The per-file authenticator key table needs a backup posture equal to the ledger's.** Losing it
  makes every affected file permanently unauditable — a new class of unrecoverable state with no
  current owner.
- **Challenge-seed unpredictability must become a stated requirement** in ADR-059/060, not an
  implicit assumption.

---

## §4 — Revised sequence

Replaces v3 §5.1. Changes are marked.

```
IMMEDIATELY — prescriptive, no reading, no council
  ├── ADR-004 addendum recording F-LTS-07 + F-LTS-08 and ratifying the fix order
  ├── ADR-012 amendment (F-LTS-09) · ADR-022 amendment (remove the cap's confidentiality claim)
  ├── ADR-072 addendum: segment_id + shard_index in the signing input      [half a session]
  ├── TestProfileConfidentialityMarginHolds — ten lines, fails on prod today, and should
  ├── Track: tags on ADR-059/060/061/065/066/067/068 (F-LTS-05); ADR ceiling → 073 (F-LTS-06)
  ├── DM constraint + partial unique index prohibiting duplicate chunk_id per provider
  ├── IC §4.1 sentence: the microservice cannot validate chunk_id; the provider enforces it
  └── Verify: fresh AONT key per segment per upload   ← closes R-60 formally

STILL OPEN, no reading required
  ├── Q66-2 — SHELBY re-derivation against Paper 37 (owned)      ← one afternoon, highest value/hour
  ├── F-LTS-11 — where authenticators live relative to Frame 1   ← blocks Proof of Storage sessions
  └── Count the mid-period handover fraction                     ← decides whether R-63 exists

M19 — LTS Foundation (needs no research)
  └── Audit every wire format for an algorithm identifier (R-61's substitution test) while the
      interfaces are open. One byte now, a version negotiation later.

THEN — CONFIDENTIALITY FIRST  [CHANGED: was third]
  R-27 → R-28 → R-29        P   confidentiality-preserving repair   ← R-28 now load-bearing
  R-17                      K   ASN feasibility + detection         ← PROMOTED from Domain E
  R-30 → R-31               K   decouple the thresholds             ← only if R-17 < 8 ASNs, or anyway

  ...and the repair-topology milestone (F-LTS-07 + F-LTS-08) runs alongside, since Domain P's
  substitution test is defined against a post-r0-gate repair frequency.

THEN — PROOF OF STORAGE
  R-47                      A′  server-side repair tagging          ← GATING; Band 0, no search
  R-50                      G   randomness beacon                   ← HARD DEPENDENCY; price drand
  R-48                      A′  stripe-level detection composition

THEN — as v3, unchanged
  R-12 → R-15               D   churn and lifetime
  R-16 · R-18 · R-19        E   correlated failure                  ← R-17 has left this domain
  R-51 → R-53               Q   seam verification    [starts now, never stops]
  R-54 → R-55               S   scale-claim validity
  R-56 → R-58               O   operational readiness
  R-23 → R-25               G   transparency logs
  R-26 · R-32 · R-33        L   single-writer election    [R-32 highest value]
  R-34 → R-37               M   local secrets
  R-38 (· R-39, reduced)    N   nonce handling
  R-59                      H   squatting note only
  R-40 → R-42               I   DoS, gaming, relay
  R-43 → R-46 (· R-63?)     J   economics, compliance, carbon
  R-61 → R-62               T   crypto agility
  R-11 · R-64               C   what survives of erasure work
```

**If only one domain gets attention: P.** Changed from v3's A′. The three-repairers proof means
every repair event discloses plaintext to *someone*, by construction, today, with no attacker —
and the operator is currently that someone.

**If only two: P and A′.** Unchanged in spirit; the coupling is now proven rather than asserted.

**If only three: add Q.** Unchanged. F-LTS-07 and F-LTS-08 are two more instances of the ADR-070
pattern — a ratified decision and its implementation diverging silently — found by reading code
against documents. Nobody has counted how many more there are.

---

## §5 — Scoreboard

| | v3 | v3.1 | Δ |
| --- | --- | --- | --- |
| Active research topics | 64 | **57** | −7 closed |
| Open council items (§7.2) | 7 | **1** | Q66-2 remains, and it is a derivation |
| Domains fully closed | — | **1** (C, except R-11/R-64) | |
| Domains reduced to a note | — | **1** (H) | |
| Domains promoted | — | **2** (P to first, G's R-50 to hard dependency) | |
| Topics promoted across domains | — | **1** (R-17, E → K, Band 2 → Band 1) | |
| New blocking findings | — | **5** (F-LTS-07 … F-LTS-11) | |
| Milestones that can close | — | **1** (Repair & Erasure Optimisation) | |

**Net:** seven topics closed, six council items cleared, one milestone retired — against five new
findings, two of which are milestones in their own right. The list got shorter and the build got
longer, which is the correct direction for a council to move things.
