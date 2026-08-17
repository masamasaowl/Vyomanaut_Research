# LTS Literature Documentation Standard — v1

**Status:** Proposed. **Track:** LTS. **Applies from:** Paper 73.
**Extends:** `style_guide.md` (which remains authoritative for demo-track notes, Papers 01–72,
and for all ADR formatting).
**Governs:** every research note produced against `reading-list.md` (LTS) from Band 0 onward.

---

## §1 — Why the demo-track format is not enough

`style_guide.md` was written from Papers 01–11. It is a good format for the job it was built for:
read a paper, extract the design pattern, name what breaks, point at an ADR. Papers 01–72 are
documented to it and should not be rewritten.

The LTS track asks a different question of the corpus. `reading-list.md` §2 audits 75 ADR files and
finds fourteen parameters across nine ADRs deferred to telemetry that does not exist, four
decisions taken by council where a published answer existed, three vendor defaults promoted to
design constants, and four load-bearing claims with no evidence behind them. The remedy that
document names is not "read more papers." It is **arrive at a number, show the substitution, and
record what would falsify it.**

The demo format cannot carry that. It has no place to put an arithmetic derivation, no way to
distinguish a claim the paper proves from a claim the note infers, and no mechanism for recording a
paper's *negative* results — the thing it rules out, which is often worth more than the thing it
proposes. `reading-list.md` §3.6 already declares that **a recorded null result is a deliverable**;
the format has nowhere to record one.

This standard adds four things and changes nothing else:

1. a **provenance discipline**, so every claim carries where it came from;
2. a mandatory **Substitution** section with working shown at Vyomanaut's real parameters;
3. a mandatory **Falsifiers** section, stating what observation would overturn the note's conclusion;
4. a **triage record**, so the decision to draft the note is itself auditable.

The positive exemplar `reading-list.md` §2.6 names — ADR-029 Addendum A's chain of *structural
assumption → named paper → substitution shown → hand check → result* — is the shape this standard
makes compulsory rather than exceptional.

---

## §2 — The provenance rule

Every factual claim in an LTS note is one of four kinds. The kind is marked inline, at the claim,
not in a bibliography at the end.

| Tag | Meaning | Example |
| --- | --- | --- |
| `[P §x]` | Stated by the paper, at the named location | `[P §4.3]` the adversary cannot learn `M` |
| `[P-num]` | A number the paper measured or published | `[P-num]` 95% of `x` values fall in 0.025–0.034 s |
| `[DERIVED]` | Arithmetic performed in this note from `[P …]` inputs plus Vyomanaut parameters. **Working must be shown.** | `[DERIVED]` 12.03 GB of peer traffic per ROTF audit |
| `[INFERRED]` | A judgement not stated by the paper and not arithmetically forced. Carries no authority on its own | `[INFERRED]` this transfers to the elected-repairer protocol |

Two rules follow, and they are the whole point of the tagging:

- **An ADR may cite `[P …]` and `[DERIVED]` claims. It may not rest on an `[INFERRED]` claim alone.**
  An `[INFERRED]` claim that a decision needs becomes an open question or a council item, not a
  premise.
- **A `[DERIVED]` claim with no visible working is a defect.** If the substitution does not fit
  inline, it goes in the Substitution section and the claim links to it.

This exists because of a specific failure the project has already had. Q66-2 records it: a formula
was lifted from Paper 37 into ADR-060 without its substitution shown, and the resulting number was
wrong twice over — arithmetically, and in what it was a number *about*. Tagging would not have
prevented the error, but it would have made the untagged step visible at review.

**Reference the location, not the page.** Section numbers, theorem numbers, figure numbers, and
equation numbers survive a change of edition; page numbers do not. Where a paper has both a
conference and a journal version, cite the one actually read and say which in the header.

---

## §3 — Required sections

An LTS note has the following sections in this order. Sections marked **required** must be present
even if the honest content is one line saying the section is empty and why.

```
# Paper NN — Full Title

  header block                                    required
  ## Provenance                                   required   (new)
  ## Problem Solved                               required
  ## Key Findings                                 required
  ## Substitution at Vyomanaut's Parameters       required   (new — promoted from optional)
  ## What This Paper Rules Out                    required   (new)
  ## Trade-offs                                   required
  ## Breaks in Our Case                           required
  ## Decisions Influenced                         required
  ## Falsifiers                                   required   (new)
  ## Disagreements                                required
  ## Corpus Delta                                 required   (new)
  ## Open Questions                               required
```

### 3.1 — Header block

```markdown
**Authors:** Name (Affiliation), Name (Affiliation)
**Venue / Year:** Venue | Year        ← the version actually read
**Also published as:** the other version, if one exists
**Topics:** #N, #N
**Track:** LTS
**Reading list:** Domain X / R-NN — Band N — the accept criterion this paper was read against
**ADRs produced:** ADR-NNN, ADR-NNN Addendum A | none
**Findings raised:** F-LTS-NN, F-LTS-NN | none
**Questions closed:** QNN-N | none
**Questions raised:** QNN-N | none
**Triage score:** N/10 (parameter reach N · trust model N · evidence N · actionability N · corpus delta N)
```

The triage score is `reading-list.md` §3.4's scorecard, recorded rather than discarded. A note
drafted at a score below 7 must say in Provenance why it was drafted anyway — Band 0 items are
drafted by instruction, not by score, and that is a legitimate reason, but it should be written down.

### 3.2 — Provenance *(new)*

Three or four lines. What was read, in what form, and what was not read.

- The exact artefact: title, venue, year, and whether it was the conference version, the journal
  version, an extended technical report, or a substitute.
- **What was not available.** Papers 66–72 already carry provenance flags of this kind (Q67-2 exists
  precisely because a substitute was used for Juels–Kaliski). Making it a section rather than a
  footnote means the gap is found by structure rather than by memory.
- Where proofs are deferred to a technical report the note did not read, say so, and mark every
  claim that depends on those proofs `[P §x, proof not read]`.

### 3.3 — Substitution at Vyomanaut's Parameters *(now required)*

The section the standard exists for. `reading-list.md` §3.3 already requires a substitution test at
triage; this makes the *result* a permanent artefact rather than a note in a triage log.

Rules:

- **Every formula the paper states that could bind a Vyomanaut parameter is evaluated at Vyomanaut's
  real values.** Not at the paper's values, not at round numbers.
- **The substitution is written out.** Symbolic form, then values, then result, then units. A reader
  must be able to re-run it without opening the paper.
- **A hand check follows any result that will be cited.** ADR-029 Addendum A's *"removing 41 is
  necessary and sufficient, deterministically. The integral returns exactly that. ✓"* is the model:
  an independent argument that arrives at the same number by a different route.
- **Where the formula does not transfer, say why, and stop.** A substitution that cannot be
  performed is a result. Record which parameter is missing and which question owns it.
- **Sensitivity, not just a point value.** Where a result depends on a quantity nobody has measured,
  give it as a table over that quantity's plausible range rather than as one number at an assumed
  value. A single number at an assumed value is how a `[VENDOR-DEFAULT]` becomes a design constant.

The standing Vyomanaut parameter set, for convenience — substitute from here, not from memory:

```
RS(16,56)        k = 16 data, r = 40 parity, n = 56, r0 = 8 (prod) / 1 (demo)
                 field GF(2^8), poly 0x11d, Vandermonde with α = 2
shard / chunk    262,144 B (256 KiB) — compile-time constant, identical in demo and prod
segment          plaintext 786,384 B under the real AONT-RS formula (not the 1.25 MB in mvp.md §7.10)
audit block      1,024 B = 64 sectors × 16 B;  256 blocks per chunk
authenticators   4,096 B per chunk = 1.5625% overhead;  field Z_p, p = 2^128 + 51
sampling         audit_sample_rate = 0.01 → c = 2,867 chunks/day for a 70 GB provider
provider         70 GB → n = 286,720 chunks;  MTTF 180–380 d (Storj NAS population — Class D)
placement        20% ASN cap;  ≥56 providers / ≥5 ASNs upload gate
background       100 KB/s budget (Blake & Rodrigues 2003 — Class D)
payments         paise-denominated integers, no floating point anywhere
```

Anything on that list carrying a Class D or `[VENDOR-DEFAULT]` warning must be named as such when it
is used, so a derived result does not launder an inherited assumption into a fresh-looking number.

### 3.4 — What This Paper Rules Out *(new)*

The negative results. This is where a two-hour read that produced no adoptable mechanism still pays
for itself.

Three things belong here:

- **Mechanisms the paper eliminates.** A proof that an approach cannot work is a permanent saving.
- **Assumptions the paper falsifies empirically.** Chen & Curtmola's demolition of the BDS
  network-delay model is worth more to Vyomanaut than either scheme they propose, because Vyomanaut
  had built the same argument (ADR-014 Defence 2) without knowing it had been measured and found
  wanting.
- **The near-miss class.** `reading-list.md` §3.3's *"Adjacent, not this"* — what this paper looked
  like it would answer and does not. Recording it stops the next reader re-running the same query.

### 3.5 — Falsifiers *(new)*

For each conclusion this note hands to an ADR: **what observation would overturn it?**

Written as a checkable statement, not a hedge. `"the margin is comfortable"` is not a falsifier.
`"if measured provider downlink exceeds 300 Mbit/s at the median, the timing separation in §4.2
inverts and this conclusion is void"` is.

Each falsifier is one of:

- **A measurement** — then it names the LaunchGate ID or the open question that owns it.
- **A proof obligation** — then it names what must be checked, against which theorem, in which paper.
- **A parameter change** — then it names the `NetworkProfile` field or ADR constant whose movement
  breaks it.

A conclusion with no falsifier is either a tautology or an `[INFERRED]` claim wearing a disguise.
Say which.

### 3.6 — Corpus Delta *(new)*

Two or three lines. What this paper adds that the corpus did not have, which existing paper it
subsumes or contradicts, and whether any earlier note now needs correction.

If it corrects an earlier note, **the correction is applied to that note in the same session**, with
a dated revision block at the top of the corrected file. The corpus is a single object, not an
append-only log; a note that is known to be wrong and left standing is worse than no note.

### 3.7 — The `≠` rule survives unchanged

`style_guide.md`'s Breaks format is canonical and is not modified:

```
[Paper assumption] ≠ [our reality]
→ [Adaptation required]
```

LTS adds one requirement: **each break is classified.**

| Class | Meaning |
| --- | --- |
| **Fatal** | The paper's result does not transfer at all. Say so plainly and move the mechanism to *What This Paper Rules Out* |
| **Costed** | Transfers, at a price. The price is quantified in Substitution |
| **Open** | Transfers if something unknown is true. That something becomes a question ID |

An unclassified break is the shape of a gap being quietly resolved, which the project's own standing
rule forbids.

---

## §4 — Revising a note in place

Papers are re-read. Band 0's fourth item is a re-derivation of Paper 37, not a new note. Handle it
like this:

- **Amend the file. Do not create `paper-37b`.** The number is a stable reference; ADRs, questions,
  and reading lists point at it.
- **Add a revision block immediately under the header**, before Provenance:

```markdown
> **Revised YYYY-MM-DD (LTS Band 0).** What changed, in one or two sentences, and which downstream
> documents were corrected as a consequence. Prior conclusions that are now withdrawn are named
> explicitly — not silently edited away.
```

- **A withdrawn conclusion is struck, not deleted.** If the original note asserted something an
  ADR relied on, the reader of that ADR must be able to find out what happened. Keep the claim,
  mark it withdrawn, state what replaced it.
- **Every document that consumed the withdrawn claim is listed** in the revision block and corrected
  in the same session, or an approval item is raised for it.

---

## §5 — When a finding becomes an ADR

The project's standing rule is *addendum over new ADR*. This standard makes the boundary testable.

| The finding… | Instrument |
| --- | --- |
| corrects a number, a citation, or the scope of an existing decision | **Addendum** to that ADR |
| falsifies a claim an ADR makes about itself | **Addendum**, and the claim is withdrawn in the ADR's own words |
| opens a decision space no existing ADR covers | **New ADR** |
| spans two ADRs and belongs wholly to neither | **New ADR**, referencing both, with a sentence in its Context saying why it is not an addendum |
| is a research direction rather than a decision | **No ADR.** It goes to the reading list domain and an open question |

That last row is load-bearing. `reading-list.md` §2.2 lists four ADRs whose entire research source is
`This review` — decisions manufactured because the format made a decision the natural output. A
finding that is not yet a decision must be allowed to remain a finding.

---

## §6 — Checklist

Before a note is considered complete:

- [ ] Header carries Track, reading-list topic, triage score, and all four ID lines (ADRs, findings, questions closed, questions raised)
- [ ] Provenance names the exact artefact and what was not available
- [ ] Every claim carries `[P §x]`, `[P-num]`, `[DERIVED]`, or `[INFERRED]`
- [ ] No `[DERIVED]` claim without visible working
- [ ] No ADR in this session rests on an `[INFERRED]` claim alone
- [ ] Substitution evaluates every bindable formula at the real parameter set, with a hand check on anything cited
- [ ] Results that depend on unmeasured quantities are given as a range, not a point
- [ ] *What This Paper Rules Out* is non-empty, or says why it is empty
- [ ] Every break is classified Fatal / Costed / Open
- [ ] Every conclusion has a falsifier naming a measurement, a proof obligation, or a parameter
- [ ] Corpus Delta names what changed, and any corrected note has a dated revision block
- [ ] Open questions checked against `open-questions.md` for duplicates before adding
- [ ] Closed questions moved to `answered-questions.md` with the answer stated, not just referenced

---

## §7 — What this standard deliberately does not do

- **It does not lengthen notes for their own sake.** A null result is three sections and half a page.
  The required sections exist so that a short note is short *for a stated reason*.
- **It does not replace the council.** `reading-list.md` §1's trigger table decides what goes to
  literature first. This standard governs what the literature produces once it gets there.
- **It does not apply to the demo track.** ADR-062 freezes the demo. Papers 01–72 keep their format.
  Backporting this standard to them would be scope expansion with no consumer.
