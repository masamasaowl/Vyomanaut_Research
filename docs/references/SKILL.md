---
name: document
description: >
  Produces a research paper summary formatted for the Vyomanaut Research docs,
  ready to save as docs/research/paper-NN-(slug).md on GitHub.
  Also produces any Architecture Decision Records (ADRs) triggered by the paper,
  ready to save as docs/decisions/ADR-NNN-(slug).md.
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

| ADR | Topic | True Status | Closed by |
|---|---|---|---|
| ADR-001 | #1 Coordination | Accepted | Papers 01–05, 07 |
| ADR-002 | #2 Proof of Storage | Accepted | Papers 05, 07 |
| ADR-003 | #3 Erasure Coding | Accepted | Papers 05, 06, 10 |
| ADR-004 | #4 Repair Protocol | Accepted | Papers 06, 09, 10 |
| ADR-005 | #5 Peer Selection | Accepted | Papers 01, 07 |
| ADR-006 | #6 Polling | Accepted | Papers 06, 08, 09 |
| ADR-007 | #7 Exit States | Accepted | Papers 08, 09 |
| ADR-008 | #8 Reliability Scoring | Accepted | Paper 08 |
| ADR-009 | #11 Background Execution | Accepted | Paper 09 |
| ADR-010 | #5 No Mobile V2 | Accepted | Papers 05, 06, 08 |
| ADR-011 | #13 Fiat Escrow | Accepted | Papers 05, 07 |
| ADR-012 | #13 Payment per Audit | Accepted | Papers 05, 07 |
| ADR-013 | #14 Consistency Model | Accepted | Paper 11 |
| ADR-014 | #19 Adversarial Defences | Accepted | Paper 07 |
| ADR-015 | #2 Audit Trail | Accepted | Papers 07, 11 |
| ADR-016 | #13 Payment DB Schema | Accepted | Paper 11 |
| ADR-017 | #2 Audit Receipt Schema | Accepted | Papers 07, 11 |
| ADR-018 | #5 Hot/Cold Bands | Accepted | Own decision |
| ADR-019 | #9 Client-Side Cipher | **Accepted** — ChaCha20/Poly1305 | Paper 17 |
| ADR-020 | #10 Key Management | **Accepted** — HKDF + pointer file | Paper 18 |
| ADR-021 | #12 P2P Transfer | **Accepted** — libp2p + QUIC v1 | Papers 13, 14 |
| ADR-022 | #15 Encoding Pipeline | **Accepted** — AONT-RS | Papers 15, 16 |
| ADR-023 | #16 Storage Engine | **Accepted** — WiscKey | Papers 25, 26, 27 |
| ADR-024 | #18 Economic Mechanism | **Proposed** — blocked on Phase 5 | — |
| ADR-025 | #1 Microservice Cluster | Accepted — (3,2,2) gossip | Paper 12 |
| ADR-026 | #17 Repair BW | **Proposed** — Hitchhiker only | — |

---

## Reading order

If the paper cannot be fully read, extract in this order:
Abstract → Conclusion → Introduction → Figures/Tables → full text.

---

## Output 1 — Research Note

**Suggested file path:** `docs/research/paper-NN-(slug).md`

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

**Suggested file path:** `docs/decisions/ADR-NNN-(slug).md`
where NNN is the next number after ADR-024.

If the paper **supersedes** an existing Accepted ADR:
1. Create the new ADR with `Supersedes: ADR-NNN`
2. Note in the research summary: `SUPERSEDES ADR-NNN` in the Decisions Influenced section
3. Remind the user to update the old ADR's `Superseded by:` field

Follow the ADR format in `docs/references/style-guide.md`.

---

## Build new Open questions

Open questions are crucial questions based on architecture, engineering or validity surrounding the project that must be answered before the build starts

Before adding any new open question, search `docs/research/open-questions.md` by keyword. A question that appears to be new may already exist under a different paper's section. Duplicates clutter the tracker.

After generating the Outputs 1 & 2, ask the user to manually add them into `open-questions.md` under a new section for this paper, with status `open` and a specific `Blocked on:` reference these questions into open-questions.md

When a question is answered by the new paper, ask the reader to update its status to `answered` in `open-questions.md` and paste the answer there.

The manual addition instruction given to the reader must be detailed, mentioning all the changes that need to be made in `open-questions.md`

---

## Quality checklist before finalising
Run the quality checklist in `docs/references/style-guide.md` before finalising.

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
1. The most important findings for the current build phase
2. Which existing ADR this paper strengthens or supersedes, if any
3. The next paper to read if the reader wants to go deeper on the most critical open question
4. Provide to the reader: Which questions in `open-questions.md` were answered and which new ones were added
