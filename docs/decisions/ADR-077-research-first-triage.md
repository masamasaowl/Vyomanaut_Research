# ADR-077 — Research-First Triage: When a Question Goes to Literature Before Council

**Status:** Accepted
**Track:** LTS
**Topic:** #19 Research Governance *(new topic)*
**Supersedes:** — *(amends `briefing-protocol.md`; adds a precondition to the `/design-council` skill)*
**Superseded by:** —
**Research source:** none — this ADR records a governance decision arising from an assumption-debt
audit of all 75 ADR files (`reading-list.md` §2). It is about *how* research enters decisions, not
about a technical mechanism.

---

## Context

The project has two instruments for closing an open design question: convene the Design Council, or
send it to the Technical Researcher. The council is fast, always available, and produces a decision
with dissent on record. Reading is slow, sometimes returns nothing, and produces a `paper-NN` before
it produces an answer.

Given that asymmetry, the council has become the default. An audit of the ADR corpus shows what that
cost:

**1. A decision was taken on a false premise that literature would have caught.** ADR-072 removed
`file_id` from the capability-token signing input on the argument that `chunk_id` is *"256 bits of
fresh, microservice-generated randomness … never reused across files."* ADR-073 established one
session later that `chunk_id` is `SHA-256(chunk_data)` — a client-computed content address. The
decision survived; the recorded argument did not. The underlying error — a capability naming its
*payload* rather than the *object* it grants authority over — is the first thing any
object-capability text addresses, and has been since Dennis & Van Horn (1966).

**2. Fourteen parameters across nine ADRs are labelled "starting value," with calibration deferred
to post-launch telemetry.** `t = 24 h` polling, `72 h` departure, `4 h` heartbeat, `5 s` gossip
window, `50 ms` background gate, release multipliers `0.95/0.80/0.65`, a `50%` vetting cap, a `90 d`
rotation cadence, and ADR-037's repair-ETA sample size — self-described as *"arbitrary but
non-trivial."* Five of these are timeout selection on a stochastic process, which is a formal
problem with a 2002 paper (Chen, Toueg & Aguilera) giving a procedure for exactly this. None of the
nine ADRs cites it, because none of them looked.

**3. Four ADRs cite `This review` as their entire research source.** ADR-065 through ADR-068 set the
launch-gate schema, the metric grammar, the alert/runbook bijection and the relay provisioning rule
between them. ADR-067's 90-day rotation cadence is a guess at a number **NIST SP 800-57 Part 1
publishes normatively.** ADR-066 invents a dimensionless-metric class the OpenMetrics convention
already handles. ADR-068's relay problem is a reorder-point problem with a textbook.

None of this is carelessness — every one of these ADRs flags its own weakness honestly, which is why
the audit was possible at all. The defect is that **honest flagging has no mechanism attached.**
"Tune after launch" is not a plan when the parameter governs whether launch survives, and nine
independent deferrals become a systemic exposure no single ADR review catches.

The counter-risk is real and must be stated: a reading-first default that applies to *everything*
would stall the project. Most questions genuinely have no literature, and the council exists because
deliberation is the right instrument for a contested tradeoff. **This ADR is not "read more." It is
a test for which of the two instruments a given question belongs to.**

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — judgement call per question** | Zero process; fastest per decision | Produced fourteen undeferred parameters and one false security argument. The judgement call is made by whoever opens the question, under time pressure, which is exactly when reading loses |
| **Reading-first by default; council only when reading returns nothing** | Maximises evidence | Stalls. Most questions have no literature, and a two-hour null-result search before every decision is a tax with no yield. Also inverts the instruments: council is *better* than reading for a genuine tradeoff |
| **A named trigger test; council remains the default where no trigger fires — chosen** | Cheap to apply (five rows, read in seconds); catches the specific classes the audit found; leaves the council as the default for what it is good at | Requires the trigger list to be maintained as new classes are discovered — accepted, and §4 assigns that |
| **Require a research source on every ADR** | Simple, mechanically checkable | Produces citation theatre. ADR-072's failure was not a missing citation; it was a missing *reading*. A mandatory field gets filled with the nearest plausible paper |

## Decision

### 1. The research-first trigger

Before a question is placed on a council docket, it is checked against the table below. **If any row
matches, the question goes to literature first, and the reading enters the council session as an
input.** The council is not skipped — it is informed.

| # | Trigger | Instances found in the audit |
| --- | --- | --- |
| **T1** | The output is a **numeric threshold on a stochastic process** | `t = 24 h`, `72 h`, `4 h`, `5 s`, `50 ms`, `0.95/0.80/0.65`, `50%`, `20 jobs` |
| **T2** | The question is a **known named problem in another field** | Relay provisioning → reorder point; timeout selection → failure-detector QoS; charge splitting → cost allocation |
| **T3** | The decision rests on a claim about a **primitive's security properties** | ADR-072's `chunk_id` unforgeability argument |
| **T4** | A **normative standard** exists | Key rotation → NIST SP 800-57; metric naming → OpenMetrics; accessibility → ADR-050 *(which did this correctly)* |
| **T5** | The number is a **vendor default promoted to a design constant** | libp2p's 128 relay reservations; three BadgerDB defaults; RocksDB's 10 MB/s rate limiter |

**T1 is the highest-yield row.** Nine of the fourteen Class A parameters match it, and one body of
theory serves five of them.

### 2. Two new ADR tags, machine-checkable

Any numeric constant in an ADR whose value is not derived carries one of:

- **`[VENDOR-DEFAULT]`** — sourced from a dependency's default. ADR-068 already does this correctly
  and unprompted: *"the 128 slots/node figure is libp2p Circuit Relay v2's **default** reservation
  limit, not a measured capacity."* That sentence is the model. Every such constant must additionally
  carry a `LaunchGate` measurement ID under ADR-065.
- **`[UNDERIVED]`** — chosen by judgement, no source. Replaces the prose formula *"starting value;
  tune empirically after launch,"* which reads as a plan and is not one. An `[UNDERIVED]` tag must
  name **either** a reading-list topic that will retire it **or** a LaunchGate ID that will measure
  it. A tag with neither fails CI.

Retro-tagging the fourteen known instances is a documentation pass, not a redesign, and is assigned
in §4.

### 3. `/design-council` gains a precondition

The skill's framing step already requires verifying that a conflict is real before convening —
*"a stale claim isn't a tradeoff, and the council isn't a substitute for checking the repo."* That
sentence is extended by one clause: **and the council isn't a substitute for checking the
literature.** The `CONFLICT:` block gains a required line:

```
RESEARCH-FIRST: [T1–T5 matched, and what was read] | [none matched — why]
```

An empty or absent line blocks the session. This is deliberately the lightest possible enforcement:
it costs one line and one honest sentence, and it makes the decision *visible* rather than implicit.

### 4. Ownership

| Task | Owner |
| --- | --- |
| Retro-tag the fourteen Class A parameters `[UNDERIVED]` and the four Class C constants `[VENDOR-DEFAULT]` | One documentation pass, before M19 closes |
| Maintain the T1–T5 trigger list as new classes surface | Whoever runs the next assumption-debt audit; recommend one per LTS milestone-map revision |
| Route matched questions to the Technical Researcher | Project lead, at docket time |

### 5. What this ADR explicitly does not do

It does not require a research source on every ADR (citation theatre — see Options). It does not
make reading the default. It does not apply retroactively to demo-track decisions, which are frozen.
And it does not treat a null search result as a failure: **a written "nothing found, here is what I
tried" satisfies the trigger and closes it**, which is what makes the test cheap enough to apply
honestly.

## Consequences

**Positive.** The five classes of error the audit found each have a mechanism attached rather than a
flag. `[UNDERIVED]` converts fourteen silent deferrals into fourteen tracked items with named
owners. The council keeps doing what it is good at — contested tradeoffs, internal inconsistencies,
and questions where the literature exists and disagrees with itself — and stops being asked to
substitute for an afternoon's reading.

**Negative.** One extra line per council session and a maintained trigger list. Some questions will
now wait two hours for a null result. Accepted: two hours against ADR-072's actual cost, which was
two ADRs, a council session, a false claim in the permanent record, and a live-verification session
spent chasing a single-shard failure that was never a single-shard failure.

**Open constraints:**

- **The trigger list is derived from one audit of one corpus.** It will be incomplete. T1–T5 cover
  every instance found in ADR-001–073; they are not claimed to be exhaustive, and the maintenance
  assignment in §4 exists because of that.
- **CI enforcement of `[UNDERIVED]`/`[VENDOR-DEFAULT]` is not specified here.** The check is
  mechanical (a tag must be followed by an `R-NN` or a LaunchGate ID) and belongs with ADR-065's
  gate tooling, but the grammar is not frozen in this ADR. → Q77-1.
- **`briefing-protocol.md` should absorb the trigger test** so the Technical Researcher and the
  council are working from one document rather than two. Left as a documentation follow-up.

## References

- `reading-list.md` §2 — the assumption-debt audit this ADR responds to; §1 carries the trigger table
- ADR-065 — LaunchGate measurement schema, the home for `[VENDOR-DEFAULT]` constants
- ADR-068 — the correct handling of a vendor default, flagged unprompted
- ADR-029 Addendum A — the positive exemplar: structural assumption → named paper → substitution
  shown → hand check → result
- ADR-072, ADR-073 — the decision and the correction that motivated T3
- Chen, Toueg & Aguilera, "On the Quality of Service of Failure Detectors" (IEEE TC, 2002) — the
  literature T1 exists to reach; Domain U, R-65
