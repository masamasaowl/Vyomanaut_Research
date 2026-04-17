# Vyomanaut Research Documentation Style Guide

> Derived from papers 1–11 in the research notes. Paper 11 (Bailis 2015) is the canonical model for a mature research note entry.
> When in doubt, mirror it.

---

## Repository structure

```
docs/
  research/
    paper-NN-<slug>.md       ← one file per paper
    open-questions.md        ← all open questions, one entry per question
    reading-list.md          ← all reading phases
    briefing-protocol.md     ← how to read papers
  decisions/
    ADR-NNN-<slug>.md        ← one file per decision
    README.md                ← ADR index
  references/
    style-guide.md           ← this file
    SKILL.md                 ← AI skill for documenting new papers
```

Every claim in a research note maps to a specific paper. Every parameter value maps to a specific ADR. Open questions live in `open-questions.md` — not inside paper files.

---

## Project context (carry into every document)

Vyomanaut V2 is a **desktop-first** distributed storage network:
- Hardened microservices for coordination (no blockchain)
- Pure P2P data transfer (libp2p + QUIC, TBD — ADR-021)
- Fiat escrow payments (Razorpay/Cashfree, India-first, UPI)
- Zero-knowledge storage (client-side encryption, service never sees plaintext)
- Reed-Solomon erasure coding (s=16, r=40, r0=8, lf=256 KB — ADR-003)
- Signed receipt exchange + Transparent Merkle Log for audit trail (ADR-015)
- Desktop providers only for V2 (MTTF 180–380 days); mobile deferred to V3 (ADR-010)
- Lazy repair, 72 h trigger threshold (ADR-004, ADR-006)
- I-confluence map: 6 ops coordinated, 14 coordination-free (ADR-013)

**19 active research topics (#1–#19).** Every paper and every ADR maps to one or more topic numbers.

---

## Research note format

File: `docs/research/paper-NN-<slug>.md`

```markdown
# Paper NN — [Full Title]

**Authors:** [Authors, Affiliation]
**Venue / Year:** [Venue | Year]
**Citations:** [count if known]
**Topics:** #N, #N
**ADRs produced:** [ADR-NNN title, or "none"]

---

## Problem Solved

[3–5 sentences. Name the exact failure mode the paper addresses.
End with one sentence on its value for this project specifically.
"Distributed storage" alone is not acceptable — name the precise gap.]

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| [Mechanism] | [Alternative] | [What this means for V2] |

---

## Breaks in Our Case

- **[Paper assumption]** ≠ **[our reality]**
  → [Adaptation required]

---

## Decisions Influenced

- **[ADR-NNN](../decisions/ADR-NNN-slug.md) [#N Topic name]** `[ACCEPTED | REVISED | SUPERSEDES ADR-NNN]`
  [Specific choice made, with parameter values.]
  *Because:* [reason from this paper]

---

## Disagreements

- **[Paper, year]:** challenges [specific claim] because [reason].
  *Implication for us:* [X]

---

## Open Questions

See [open-questions.md](open-questions.md) — questions QNN-N through QNN-N.
```

**For papers with quantitative results or key quoted passages** (like Papers 10 and 11), add a `## Key Findings` section between Problem Solved and Trade-offs, and a `## Engineering Questions` section at the end.

---

## ADR format

File: `docs/decisions/ADR-NNN-<slug>.md`

```markdown
# ADR-NNN — [Short imperative title, ≤50 chars]

**Status:** Accepted | Proposed | Superseded by ADR-NNN | Deprecated
**Topic:** #N [Topic name]
**Supersedes:** — | ADR-NNN
**Superseded by:** — | ADR-NNN
**Research source:** Paper NN — [title]

---

## Context

[What forced this decision? 2–4 sentences.
Name the failure mode this decision prevents.]

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| [Option A — chosen] | | |
| [Option B — rejected] | | |

## Decision

[The exact choice made. Include parameter values.
One paragraph. Specific enough that a new engineer can implement without asking questions.]

## Consequences

**Positive:**
- [What becomes easier]

**Negative / trade-offs:**
- [What becomes harder]

**Open constraints:**
- [What must remain true for this decision to hold]

## References

- [Paper NN — title](../research/paper-NN-slug.md)
- [ADR-NNN — related decision](ADR-NNN-slug.md)
```

---

## Section-by-section rules

### Problem Solved
- 3–5 sentences maximum
- Must be specific — name a failure mode, not just a domain
- Must end with one sentence on value for Vyomanaut V2 specifically

### Trade-offs
- Be honest about what the paper gives up, not just what it gains
- Use the table format — one row per trade-off

### Breaks in Our Case
- Use the `≠` notation exactly as specified — this is a searchable pattern
- Every break must end with `→ [Adaptation required]`
- This section is where intellectual honesty lives

### Decisions Influenced
- Name the ADR number explicitly every time
- Include parameter values where they exist
- If the paper supersedes an earlier decision, state it: `SUPERSEDES ADR-NNN`

### Open Questions
- Questions do not live inside paper files — they live in `open-questions.md`
- The paper file just points to the relevant question IDs
- New questions must be checked against `open-questions.md` first — do not create duplicates

### ADR status lifecycle
```
Proposed → Accepted → (if revised) Superseded by ADR-NNN
                    → (if obsolete) Deprecated
```
A `Proposed` ADR is always blocked on something explicit. A `Superseded` ADR is never deleted — it stays as a record of what was decided and why it changed.

---

## The ≠ rule (canonical)

Every Breaks section must use the exact format:
```
[Paper assumption] ≠ [our reality]
→ [Adaptation]
```
This is searchable. Future contributors will grep for `≠` to find where the system diverges from its theoretical foundations.

---

## Tone and language rules

- Write in second person when addressing the builder: "your system", "your audit log"
- Use present tense for current decisions; past tense for paper history
- Avoid: "it is worth noting", "importantly", "it is interesting that"
- Prefer: direct statements
- Numbers always in concrete units: "39 Kbps/peer", not "low bandwidth"
- No emojis

---

## Length calibration

| Paper type | Research note | ADRs produced |
|---|---|---|
| Survey / SoK | Long — extract every major design pattern | 2–4 ADRs |
| Measurement / empirical | Medium — focus on numbers that feed parameters | 1–2 ADRs |
| Mathematical formalism | Long — derive formulas, apply V2 values | 1–3 ADRs |
| Short protocol paper | Medium — state machine and constants | 1 ADR |
| RFC / specification | Medium — connection state machine and error cases | 1 ADR |
| Whitepaper | Short — 2–4 decisions max | 1–2 ADRs |
