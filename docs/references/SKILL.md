---
name: design-council
description: >
  Convenes a five-lens internal design council to resolve genuine architecture or
  implementation-strategy conflicts on Vyomanaut V2 — two governing docs disagreeing, a
  contributor's audit contradicting the live implementation, an open planning decision, or a
  BUILD BLOCKER that doesn't resolve from IC/DM/ARCH alone. MANDATORY TRIGGERS: "council this",
  "run the council", "war-room this", "pressure-test this design", "stress-test this
  architecture". STRONG TRIGGERS: "should we do X or Y" for a real tradeoff, "which approach
  for [subsystem]", a contributor review disagreeing with the current implementation, an
  unresolved planning decision from a prior session. Do NOT trigger for single-answer doc
  lookups, compile errors, or routine bugs — route those to the build skill. Not for
  plan-completeness interviews (critique skill) or writing up an already-settled decision
  (document skill).
---

# Design Council — Vyomanaut V2

Left alone, Claude tends to resolve a conflict from whichever angle it reaches for first. Fine
for a bug; a liability for a decision that shapes the data model, the ledger, or provider
trust and outlives the session. The council runs one conflict through five deliberately
different reasoning passes, cross-critiques them anonymously, then synthesizes a verdict —
not "it depends," an actual call, with the dissent on record.

The council decides *what and why*. It doesn't write Go or draft docs — hand the verdict to
**build** to implement, or to **document** for an ADR, only if asked.

## When to convene

Good: two docs (or a doc and the live code) genuinely disagree; a contributor's finding
conflicts with the implementation; an open planning decision needs closing; a design choice
has real tradeoffs at prod scale that aren't obvious at demo scale.

Skip it: the answer is one grep or one doc section away; it's a compile error or a style
choice; you haven't yet verified whether the "conflict" is even real. Verify first — a stale
claim isn't a tradeoff, and the council isn't a substitute for checking the repo.

## How it runs

**1. Frame the conflict.** Pull what's relevant — IC/DM/ARCH/REQ/MVP/OAS, prior ADRs, the live
repo/DB. Check the code before trusting a doc claim; ask for a section rather than guessing if
something key is missing. Write one neutral paragraph:

```
CONFLICT: [position A] vs. [position B]
AFFECTED: [subsystem / table / milestone]
CONSTRAINTS: [relevant NetworkProfile fields, import rules, CI gates]
AT STAKE: [why getting it wrong is costly]
```

If Aryan already leans one way, note that separately — don't let it shape what the seats see.

**2. Five seats take a position.** Each seat may draw on whatever actually supports its case —
IC/DM/ARCH, the live repo, prior ADRs, or general distributed-systems judgment. None of that
is mandatory reading and no seat has to cite a doc section if the argument doesn't need one;
treating the docs as the only valid evidence just anchors every seat to what the docs already
assume, which is a problem exactly when the docs themselves are what's in question.

- **Adversary** — hunts for the race, replay, or constraint violation each option lets through.
- **Systems Theorist** — asks what invariant is actually at stake; may conclude the conflict is
  really a gap in DM/REQ rather than a code question.
- **Scale Advocate** — checks the choice against prod profile (16/56 shards, thousands of
  providers), not just demo.
- **Outsider** — reads only the framed conflict, no session history; flags anything that only
  makes sense with tribal knowledge nobody wrote down.
- **Implementer** — checks it against import constraints, forbidden patterns, and whether it
  clears CI this week.

If subagents are available, spawn all five in parallel with only the framed conflict — nothing
else. Without subagents, write each seat as a genuinely separate pass: commit to that seat's
starting assumption before writing it, and don't let a later seat soften into an earlier one's
conclusion just because it was written second. 100–200 words each.

**3. Peer review.** Anonymize the five positions (A–E, shuffled — no fixed mapping to seat
order). Each seat reviews all five: strongest position and why, biggest blind spot and where,
what all five missed. Under 150 words each.

**4. Chairman's verdict.** One synthesis pass, structured as:

```
## Where the Council Agrees
## Where the Council Clashes
## Blind Spots the Council Caught
## The Recommendation
## What This Means for the Build
```

The chairman can side with a minority seat if its reasoning is stronger — say why. If all five
converged, say so, but check it's not five passes defaulting to the same safe-sounding answer
for weak reasons rather than genuine agreement.

**5. Present.** Post the verdict in chat as `## Design Council Verdict — {title}`, markdown
only. If it's worth preserving — it overrides a doc, settles an open decision, or changes an
invariant — offer to hand it to build or document. Don't draft either unprompted.

## Bias guardrails

- **Framing bias** — the CONFLICT/AT STAKE paragraph goes to all five seats verbatim; keep it
  neutral, and keep any stated preference of Aryan's out of it.
- **Resource anchoring** — seats are made aware of IC/DM/ARCH/ADRs/the repo, never required to
  use only those; the strongest argument wins, not the most-cited one.
- **Order/recency bias** — in sequential execution (no subagents), a later seat can drift
  toward agreeing with earlier ones just from proximity; write each from a fresh starting
  assumption, and have the chairman weigh argument strength, not word count or write order.
- **Single-reasoner bias** — five seats voiced by one model isn't five independent minds; the
  peer-review step exists specifically to catch shared blind spots a single pass wouldn't
  notice on itself.
- **False consensus** — agreement across all five is a signal, not proof; the chairman should
  say explicitly when convergence looks earned versus when it looks like everyone defaulted to
  the same instinct.
