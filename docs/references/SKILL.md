---
name: document
description: >
  Produces a research paper summary formatted for the Vyomanaut Research docs,
  ready to save as docs/research/paper-NN-<slug>.md on GitHub.
  Also produces any Architecture Decision Records (ADRs) triggered by the paper,
  ready to save as docs/decisions/ADR-NNN-<slug>.md.
  Trigger when a paper PDF or text is provided and the user wants it documented,
  summarised, or added to the research log. Also trigger on "document this",
  "write up the paper", "add this to the readme", "summarise for the project".
  Output is plain markdown — NO java/code fences around prose.
---

# Document Skill — Vyomanaut Research

Produces two outputs per paper:
1. A **research note** (`docs/research/paper-NN-slug.md`)
2. Zero or more **Architecture Decision Records** (`docs/decisions/ADR-NNN-slug.md`)

---

## Before writing anything

Read the style guide at `docs/references/style-guide.md` for format rules.

Then carry this project context into every summary:

**Vyomanaut V2 — current architecture:**
- Hardened microservices for coordination; no blockchain
- Pure P2P data transfer via libp2p + QUIC (TBD — ADR-021)
- Fiat escrow: Razorpay/Cashfree, India-first, UPI
- Zero-knowledge storage — client-side encryption, service never sees plaintext
- Reed-Solomon erasure coding: s=16, r=40, r0=8, lf=256 KB (ADR-003)
- Signed receipt exchange + Transparent Merkle Log for audit trail (ADR-015)
- Desktop providers only for V2 (MTTF 180–380 days); mobile deferred to V3 (ADR-010)
- Lazy repair, 72 h trigger threshold (ADR-004, ADR-006)
- I-confluence map: 6 ops coordinated, 14 coordination-free (ADR-013)

**19 active research topics (#1–#19).**

**Current ADR index (check before writing DECISIONS INFLUENCED — do not re-open accepted ADRs without a supersession):**

| ADR | Topic | Status | Title |
|---|---|---|---|
| ADR-001 | #1 | Accepted | Microservices + Kademlia DHT for coordination |
| ADR-002 | #2 | Accepted | PoR Merkle challenge (continuous + transitory) |
| ADR-003 | #3 | Accepted | RS erasure coding: s=16, r=40, r0=8, lf=256 KB |
| ADR-004 | #4 | Accepted | Lazy repair, r0=8, 72 h trigger |
| ADR-005 | #5 | Accepted | Storj 4-subsystem reputation pipeline |
| ADR-006 | #6 | Accepted | Polling: 24 h; departure threshold: 72 h |
| ADR-007 | #7 | Accepted | 4 exit states: temp / promised / silent / announced |
| ADR-008 | #8 | Accepted | 3-window rolling score: 24 h / 7 d / 30 d |
| ADR-009 | #11 | Accepted | Desktop-only V2; ≤5% CPU for background tasks |
| ADR-010 | #5 | Accepted | No mobile providers in V2; deferred to V3 |
| ADR-011 | #13 | Accepted | Fiat escrow; UPI via Razorpay/Cashfree; India-first |
| ADR-012 | #13 | Accepted | Payment per audit passed, not per GB stored |
| ADR-013 | #14 | Accepted | I-confluence map: 6 coordinated, 14 free |
| ADR-014 | #19 | Accepted | 4 adversarial defences: Geppetto / outsourcing / generation / JIT |
| ADR-015 | #2 | Accepted | Signed receipt exchange + Transparent Merkle Log |
| ADR-016 | #13 | Accepted | PN-counter CRDT; paise integer; idempotency key UUID |
| ADR-017 | #2 | Accepted | 12-field audit receipt with Ed25519 dual signatures |
| ADR-018 | #5 | Accepted | Explicit hot/cold provider bands (Cold only in V2) |
| ADR-019 | #9 | Proposed | Client-side encryption — blocked on Szabó 2025 |
| ADR-020 | #10 | Proposed | Key management — deferred to Phase 3 |
| ADR-021 | #12 | Proposed | P2P transfer protocol — blocked on libp2p spec + RFC 9000 |
| ADR-022 | #15 | Proposed | Encrypt-then-code vs code-then-encrypt — blocked on Szabó 2025 |
| ADR-023 | #16 | Proposed | Provider-side storage engine — deferred to Phase 2C |
| ADR-024 | #18 | Proposed | Economic mechanism design — deferred to Phase 5 |

---

## Reading order

If the paper cannot be fully read, extract in this order:
Abstract → Conclusion → Introduction → Figures/Tables → full text.

---

## Output 1 — Research Note

**Suggested file path:** `docs/research/paper-NN-<slug>.md`

Follow the format in `docs/references/style-guide.md` exactly.

Key rules:
- No `java` or any other code fences around prose
- Open questions go in `docs/research/open-questions.md`, not in the paper file — the paper file just references the question IDs
- Every break uses the `≠` notation with `→` adaptation
- Every decision references an ADR number
- If this paper closes a question in `open-questions.md`, note the question ID in Decisions Influenced and mark it answered in the tracker

---

## Output 2 — Architecture Decision Record(s)

Produce one ADR per decision this paper **closes or revises**. Do not produce an ADR for a decision that is still open.

**Suggested file path:** `docs/decisions/ADR-NNN-<slug>.md`
where NNN is the next number after ADR-024.

If the paper **supersedes** an existing Accepted ADR:
1. Create the new ADR with `Supersedes: ADR-NNN`
2. Note in the research summary: `SUPERSEDES ADR-NNN` in the Decisions Influenced section
3. Remind the user to update the old ADR's `Superseded by:` field

Follow the ADR format in `docs/references/style-guide.md`.

---

## Before checking open questions

Before adding any new open question, search `docs/research/open-questions.md` by keyword. A question that appears to be new may already exist under a different paper's section. Duplicates clutter the tracker.

When a question is answered by the new paper: update its status to `answered` in `open-questions.md` and add the answer.

When a new question arises from the paper: add it to `open-questions.md` under a new section for this paper, with status `open` and a specific `Blocked on:` reference.

---

## Quality checklist before finalising

**Research note:**
- [ ] Suggested file path given at top
- [ ] Problem Solved is specific — names a failure mode, not just a domain
- [ ] Every "Breaks" entry uses `≠` with `→` adaptation
- [ ] Decisions Influenced reference ADR numbers with status tags
- [ ] Open Questions section points to question IDs in open-questions.md
- [ ] No open questions duplicated from previous papers

**ADR(s):**
- [ ] Title is ≤50 chars, imperative mood
- [ ] Status is set correctly
- [ ] If superseding an existing ADR — the old number is referenced
- [ ] Options Considered has at least 2 options with honest cons
- [ ] Consequences names at least one negative trade-off
- [ ] Open constraints section states what must remain true

---

## Length calibration

| Paper type | Research note | ADRs |
|---|---|---|
| Survey / SoK | Long — 6–8 trade-offs, full pattern extraction | 2–4 ADRs |
| Measurement / empirical | Medium — numbers that feed parameters | 1–2 ADRs |
| Mathematical formalism | Long — derive formulas, apply V2 values | 1–3 ADRs |
| Short protocol paper | Medium — state machine and constants | 1 ADR |
| RFC / specification | Medium — connection state machine and error cases | 1 ADR |
| Whitepaper | Short — 2–4 decisions max | 1–2 ADRs |

---

## Closing note to the developer

After presenting both outputs, add one paragraph (outside all code blocks) stating:
1. The single most important finding for the current build phase
2. Which existing ADR this paper strengthens or supersedes, if any
3. The next paper to read if the reader wants to go deeper on the most critical open question
4. Which questions in `open-questions.md` were answered and which new ones were added
