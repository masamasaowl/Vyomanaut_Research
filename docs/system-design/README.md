# Universal Rules
 
These rules apply to every document inside within the system design repository without exception.
 
### 0.1 Hyperlink discipline
 
Every reference to another document in the repository must be a relative hyperlink.
Every reference to a paper must link to its research note in `docs/research/`. Every
ADR citation must link to its file in `docs/decisions/`. Every FR/NFR citation must
link to the relevant section of `docs/system-design/requirements.md`.
 
**Accepted form:** `[ADR-027](../decisions/ADR-027-cluster-audit-secret.md)`  
**Rejected form:** "see ADR-027" or "ADR-027" without a link
 
The only exception is in Mermaid diagram source blocks, where hyperlinks cannot be
embedded. In those blocks, use the short form (e.g. `ADR-027`) and provide a
cross-reference table immediately after the diagram block.
 
### 0.2 Source of truth hierarchy
 
When this specification conflicts with an ADR, the ADR wins. When an ADR conflicts
with `architecture.md`, the ADR wins. When `requirements.md` conflicts with
`architecture.md`, `requirements.md` wins. The new documents must not introduce
new decisions — they document decisions already made.
 
### 0.3 No new scope
 
None of the documents below may introduce new design decisions, new component names,
new API endpoints, new database columns, or new error codes that are not already
defined in an accepted ADR, `architecture.md`, `requirements.md`, `data-model.md`,
or `openapi.yaml`. If a gap is discovered, stop authoring, file it in
`docs/research/open-questions.md`, and note it in the document with a `> TODO:` callout.
 
### 0.4 Integer paise and no float
 
Any document that mentions money must use integer paise (not rupees, not decimals).
This mirrors [Invariant 4 in `data-model.md`](./data-model.md#3-design-invariants) and
[NFR-046 in `requirements.md`](./requirements.md#7-non-functional-requirements).
 
### 0.5 Diagram format
 
All sequence diagrams use [Mermaid `sequenceDiagram`](https://mermaid.js.org/syntax/sequenceDiagram.html)
syntax, rendered in GitHub Markdown. All state diagrams use `stateDiagram-v2`. All
ER-style or flow diagrams use `flowchart LR` or `flowchart TD`. No PlantUML. No
image files. Source must be in the Markdown file itself so it is diffable.
 
Each diagram must have:
1. A title comment `%% Title` at the top of the Mermaid block.
2. A numbered cross-reference table after the block mapping step labels to ADRs/FRs.
3. A prose paragraph before the block explaining what the diagram shows and why it matters.
4. A "What this diagram does not show" note — explicitly bounding scope prevents
   the reader from inferring unintended behaviour from a simplified diagram.
### 0.6 Failure handling is always explicit
 
Every sequence diagram must have at least one `alt` or `opt` block showing the
primary failure path for that flow. Diagrams that show only the happy path are
incomplete and will be rejected at review.
 
### 0.7 Versioning
 
Each document opens with a metadata block:
 
```markdown
**Status:** Draft | Review | Accepted  
**Version:** 1.0  
**Date:** [ISO 8601]  
**Authors:** Vyomanaut Engineering  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
**Supersedes:** —  
**Companion documents:** [list with hyperlinks]
```
 