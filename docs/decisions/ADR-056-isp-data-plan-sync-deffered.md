# ADR-056 — Defer ISP-API Sync; Ship Client-Side Data-Plan Estimate

**Status:** Proposed — blocked on external regulatory rollout (see Open constraints)
**Topic:** #20 Client-Facing UX & Copy
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 57

---

## Context

The provider dashboard is proposed to warn when stored data risks exceeding a provider's remaining ISP data plan, on the reasoning that data-plan availability matters more than disk-space availability for a provider running on a metered or capped connection. This requires knowing the provider's remaining plan balance. Paper 57 finds that no regulated, standardized API pathway exists in India today for a third-party app to query that balance with user consent — DEPA's Consent Manager framework is live for the financial sector only; telecom is explicitly a future sector on the roadmap, with no committed date.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Wait for DEPA telecom rollout before building any version of this feature | No wasted engineering effort on an interim design | No committed rollout date exists; could mean shipping nothing on this front for years |
| Pursue bilateral API deals with individual telecom operators | Could technically work sooner than a regulatory rollout | Disproportionate business-development effort for a V3 storage feature; operator-specific, not reusable; out of scope for this research |
| **Client-side estimate: track this app's own upload/download via OS network counters; combine with a manually-entered plan size and reset date** | Buildable now, no external dependency; degrades gracefully (works even if the manual entry is skipped, just with less precision) | Cannot see plan usage from *other* apps on the same connection — the estimate is a lower bound on true consumption, not a true remaining-balance figure |

## Decision

Ship the **client-side estimate** as the V3 interim design, not a true ISP sync:

- The provider daemon tracks its own cumulative upload and download volume using OS-level network interface accounting (already necessary infrastructure for bandwidth-based operational limits elsewhere in the system).
- The provider app asks the user to optionally enter their known plan size (e.g., "1.5 GB/day," "40 GB/month") and reset date. This is a manual input, not a query — no ISP integration is attempted.
- The dashboard computes a **lower-bound estimate** of remaining plan headroom: `entered_plan_size − (this app's tracked usage since reset date)`. The UI must communicate that this is Vyomanaut's own usage only, not total connection usage, so a provider running other bandwidth-heavy applications is not misled into believing more headroom remains than actually does.
- **Priority framing carries over unchanged:** the proposal's core point — that data-plan availability matters more than disk-space availability for a metered connection — still holds and still governs how warnings are surfaced; only the data source behind the warning changes from "queried" to "estimated."
- This is explicitly an interim design. Revisit true API sync if DEPA's telecom Consent Manager sector goes live (see Open constraints).

## Consequences

**Positive:**

- Buildable immediately, with no dependency on external regulatory timelines
- Fails gracefully: even a provider who skips the manual plan-size entry still gets a usage figure (just not a headroom warning) rather than a broken feature
- No new trust boundary or third-party data-sharing relationship is introduced

**Negative / trade-offs:**

- The estimate is provably incomplete — it cannot see usage from any other application, so "remaining headroom" is always an overestimate relative to the provider's true remaining plan
- Puts a data-entry burden on the provider (plan size, reset date) that a true API integration would not require
- Reopens if DEPA's telecom rollout happens; this design has a known, if undated, expiration

**Open constraints:**

- Blocked on nothing internally — buildable now. The *upgrade path* (true API sync) is blocked on an external regulatory event (DEPA telecom Consent Manager rollout) with no committed date, per Paper 57
- OS-level network accounting method differs by platform (Windows, macOS, Linux) and is not specified in this ADR — implementation detail for the build phase

## References

- [Paper 57 — DEPA and Account Aggregator framework](../research/paper-57-depa-account-aggregator.md): source of the regulatory-availability finding this decision is built on

---
