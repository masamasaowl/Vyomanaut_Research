# Paper 43 — Condor: A Hunter of Idle Workstations

**Authors:** Michael J. Litzkow, Miron Livny, Matt W. Mutka (University of Wisconsin–Madison)
**Venue / Year:** 8th International Conference on Distributed Computing Systems (ICDCS) | 1988, pp. 104–111
**Citations:** not independently verified this session; winner, 2020 IEEE TCDP ICDCS High-Impact Paper Award — a strong independent signal of sustained influence
**Topics:** #21
**ADRs produced:** none — strengthens ADR-009; feeds the forthcoming provider-audience-segmentation decision

---

## Problem Solved

In 1988, a university or company already had more idle workstation capacity, in aggregate, than any single department could otherwise access — a professor's machine sits untouched while she teaches; a lab of forty machines is nearly empty overnight. Condor finds that idle capacity, runs background batch jobs on it without disturbing the owner, and migrates the job elsewhere the instant the owner returns. It is the founding system for the institution-led half of your provider strategy: turning an organization's own existing desktops into a shared resource, administered by the organization itself rather than opted into by individual strangers.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Absolute, instantaneous owner priority — any input activity evicts the visiting job immediately | Negotiated or delayed eviction | Maximizes owner goodwill (never contested for their own machine) at the cost of job/work loss on every eviction, acceptable for cheaply-restartable batch compute |
| A central matchmaker collecting machine/job "ads" | Peer-to-peer negotiation between jobs and machines | Works well when resource donors are institutionally organized and already known to each other; does not scale to anonymous, adversarial participants the way BOINC's model does |
| Institution-led, IT-administered deployment | Individual, self-serve opt-in | Yields high machine density per adoption event, at the cost of requiring an actual organizational relationship before any machine runs the software |

---

## Breaks in Our Case

- **Condor's original (1988) central manager ran with elevated/root privileges to impersonate the submitting user on remote machines**
  ≠ **your provider daemon runs as an ordinary, unprivileged desktop application with no cross-user impersonation**
  → Treat Condor's original privilege model as a documented weakness to avoid, not a pattern to imitate — see Disagreements below.

- **Condor migrates jobs on *any* input activity, including from a user other than the one who configured the pool**
  ≠ **your provider daemon runs continuously regardless of interactive use, gated only by OS-level idle/AC-power state (ADR-009), not per-keystroke eviction**
  → Condor's aggressive, instant eviction is nearly free for a batch compute job. Serving a stored chunk is a rare, brief event for you — nothing this aggressive is needed or wanted.

- **Condor pools are deployed and administered top-down by an institution's own IT/research-computing staff**
  ≠ **you have not yet decided whether your institutional route is IT-deployed or individually opted into on institutionally-owned machines**
  → Per `ux-decisions.md` §5.2, this is a deliberate, still-open business-development choice, not one this paper settles for you.

---

## Decisions Influenced

- **[ADR-009](../decisions/ADR-009-background-execution.md) [#11 Background Execution]** `ACCEPTED — independently confirmed`
  A second, independent, 36-years-earlier precedent for the same owner-always-wins idle-detection principle already encoded in ADR-009 — this time from an institutional-compute context rather than BOINC's individual-volunteer one.
  *Because:* the paper documents this as the design that made Condor viable in practice, not an incidental detail.

- **Provider-audience-segmentation decision (not yet an ADR — forthcoming, per `ux-decisions.md` §5.2)**
  Condor is the primary historical evidence that an institution-led, IT-administered rollout of idle-desktop harvesting is a real, mature model — separate from and requiring different first steps than an individual opt-in model (Paper 42).
  *Because:* Condor pools have always been organizationally deployed, continuously since 1988, and continue today as HTCondor "at thousands of campuses, labs, and organizations."

- **App least-privilege design principle for the Provider app (not yet an ADR — forthcoming)**
  The Provider app should require the minimum OS privilege an ordinary desktop app needs, nothing resembling Condor's original privileged central-manager design.
  *Because:* see Disagreements — an independent patent filing identifies Condor's root-privilege requirement as its principal security weakness.

---

## Disagreements

- **US Patent 5,978,829** (a competing resource-sharing system, filed 1997): directly challenges Condor's original security model, stating that its centralized scheduler "requires that the server have root privileges in order for it to create processes that impersonate the individual users who submit jobs to it," and that any coding flaw in the server "may allow someone other than the authorized user to gain access to other users' privileges." The same filing also criticizes Condor's all-or-nothing eviction as causing "needless migration and its concomitant work lossage" when a user other than the primary owner briefly touches a shared machine.
  *Implication for us:* both critiques are taken as direct design cautions above. Later Condor/HTCondor versions have evolved considerably past the 1988 architecture this paper describes — the caution is about the pattern, not a claim that HTCondor is unsafe today.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q43-1.
