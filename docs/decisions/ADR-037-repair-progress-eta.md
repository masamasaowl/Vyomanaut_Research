# ADR-037 — Repair Progress: Qualitative Signal in V1, Numeric ETA Deferred to Empirical Data

**Status:** Proposed
**Topic:** #20 Client-Facing UX & Copy (see ADR-034)
**Supersedes:** —
**Superseded by:** —
**Research source:** requirements.md FR-019, §6.2 ("Degraded file state"); `internal/api/owner.go` (Availability computation); `internal/repair/queue.go`, `internal/repair/executor.go`; `migrations/001_initial_schema.sql` (`repair_jobs`); `Vyomanaut_V2` @ `55467523`

---

## Context

§6.2 commits to a specific promise for a degraded file: *"Show which file is degraded, explain in plain language that repair is in progress, and give an estimated completion time."* IC §14.2 (this session) confirms the underlying three-tier signal (`OK`/`DEGRADED`/`CRITICAL`) already exists and maps cleanly onto FR-019's "available / degraded / unavailable" language. The **estimated completion time** does not exist anywhere: no query, computation, or stored figure in `internal/repair` converts a job's queue position into a duration.

The queue itself has real structure to build on: `repair_jobs.priority` (`EMERGENCY` / `PERMANENT_DEPARTURE` / `PRE_WARNING`, `internal/repair/queue.go`) determines processing order, and jobs are executed one at a time by `ExecuteRepairJob` (`internal/repair/executor.go`) — download surviving shards, reconstruct, upload the replacement. Critically, `repair_jobs` (`migrations/001_initial_schema.sql`) already records `created_at`, `started_at`, and `completed_at` on every job. **The raw data needed to eventually compute a real duration estimate is already being captured — nothing currently aggregates it into a rate or percentile.**

That distinction matters for the decision: this isn't a case where the schema is missing a field (contrast §2.2/ADR-016's held-vs-pending earnings gap, where the ledger genuinely cannot represent the fact needed). Here, the fact will exist the moment jobs start completing in a live network — there's simply no history yet to compute from, because the network hasn't launched. Fabricating a number now (e.g., a hardcoded "usually completes within 2 hours") would be asserting a precision the product has no basis for, at exactly the moment §6.2's own instructions warn against showing raw technical detail the user can't act on — a confidently wrong ETA is arguably worse than showing none.

## Decision

**V1 ships the qualitative three-tier signal (IC §14.2) without a numeric ETA.** A degraded/critical file shows: *"Repair in progress"* (`DEGRADED`) or *"Emergency repair in progress"* (`CRITICAL`) — no duration claim.

**Instrumentation ships alongside v1** so a real estimate becomes possible later without a second data-model change:

1. A materialised view or scheduled query computes, per `priority` tier, the p50/p90 of `completed_at − started_at` (execution duration) and, separately, `started_at − created_at` (queue wait) over a trailing window (30 days, matching the escrow rolling-window precedent in ADR-024 for consistency of "recent behavior" framing across the system).
2. Once a tier has accumulated a minimum sample size (proposed: 20 completed jobs — arbitrary but non-trivial; tune post-launch), its p90 duration becomes available to expose.
3. A follow-up decision (amend this ADR, not a new one) sets the actual exposure rule once real numbers exist — e.g., "show a range, not a point estimate" is likely the right eventual answer, but is premature to lock in before any real distribution has been observed.

**§6.2's edge-state copy is amended** to read: *"Show which file is degraded, explain in plain language that repair is in progress, and — once sufficient completed-repair history exists to estimate a range — give an estimated completion time."* The v1 qualitative-only behavior is a deliberate, temporary scope reduction, not a missed requirement.

## Alternatives considered

- **A — Ship a hardcoded or hand-guessed estimate now (e.g., "usually within a few hours").** Rejected: no data exists to support any specific number; asserting one anyway risks being confidently wrong in the exact edge case (`CRITICAL`, a file at risk) where trust matters most.
- **B — Estimate from queue depth alone (e.g., "3 jobs ahead of yours"), no time unit.** Considered a viable middle ground — it's honest (a raw count, not a fabricated duration) and buildable from data that already exists (`COUNT(*) WHERE priority = X AND status = 'QUEUED' AND created_at < this job's created_at`). Not adopted as the v1 default because a bare position number ("3 jobs ahead") requires the user to already understand queue-based processing to interpret it, and §6.2 specifically asks for something in plain language — but this is the natural first upgrade once instrumentation data is thin, before a full duration estimate is trustworthy, and worth revisiting as an intermediate step.
- **C — Block M15's degraded-file UI entirely until a numeric ETA is ready.** Rejected: the qualitative signal alone is still strictly more informative than §6.2's fallback "do not show a blank screen" floor, and there is no reason retrieval-status clarity should wait on a launch-history-dependent feature.

## Consequences

**Positive:** the file-list UI (M15) is not blocked on data that cannot exist before launch; the instrumentation decided here means a future numeric (or ranged) estimate is a query/exposure change, not another schema migration; avoids shipping a number with no empirical basis.

**Negative / trade-offs:** v1 data owners with a degraded file get less specific reassurance than §6.2 originally promised, for an unspecified period after launch (until the sample-size threshold in point 2 is met per tier).

**Affected:** `requirements.md` §6.2 (copy amendment above); `internal/repair` (new aggregation query/view — additive, no schema change); IC §14.2 (already notes this ADR); M15 client design (renders qualitative labels only, until a follow-up amendment says otherwise).

## Validation

Once live for ~30 days post-launch: confirm each priority tier has reached the minimum sample size in Decision point 2, and that the resulting p50/p90 figures are stable enough (not swinging wildly week to week) to be worth exposing at all — if they're too noisy even at that point, the right move may be extending the trailing window before amending this ADR to expose a number.
