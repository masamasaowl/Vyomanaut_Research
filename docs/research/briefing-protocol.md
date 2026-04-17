# Research Briefing Protocol

How we would be documenting this project
---

## Step 1 — Start with a survey paper

When entering a new topic, start with a survey, not an original paper.
Surveys summarise the landscape effectively before you invest time in primary sources.

## Step 2 — Read papers in order

```
(start)  Abstract → Conclusion
(if still relevant) → Introduction → Figures & Diagrams
(if still relevant) → read entire paper
```

## Step 3 — Use citations in both directions

Visualise the connected papers. Follow both the papers a work cites and the papers that cite it.

## Step 4 — For real systems, prefer primary sources

| Source type | Examples |
|---|---|
| System papers | USENIX |
| Conferences | OSDI, SOSP, EuroSys |
| Engineering blogs | Company engineering blogs |
| Protocol specifications | IETF RFCs |

## Step 5 — Don't memorise — ask three questions

1. What problem does it solve?
2. What trade-offs did it accept?
3. What would break in my context at scale?

## Step 6 — Read implementation after reading the paper

Example: after reading Kademlia, study how BitTorrent or libp2p implements it.

## Step 7 — Before finalising, search for disagreements

Search for:
1. Why the technology failed
2. Commonly reported problems
3. What all the alternatives are

This helps defend a decision.

---

## Research note template

For reference — the full format is in [`docs/references/style-guide.md`](../references/style-guide.md)
and the AI skill that produces it is in [`docs/references/SKILL.md`](../references/SKILL.md).
