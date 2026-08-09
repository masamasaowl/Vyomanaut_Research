# ADR-061 — Flat Per-Shard Split for Charge Distribution

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (charge-time distribution)
**Supersedes:** —
**Superseded by:** —
**Research source:** M10 contributor audit review (`internal/payment` findings #1, #8); two
design-council sessions (Aug 2026); `Vyomanaut_V2` checkout inspected directly for this ADR
(`internal/payment/release.go`, `internal/payment/ledger.go`, `internal/api/owner.go`,
`internal/repair/departure.go`); data-model.md §4.3–4.9; ADR-012; ADR-016; ADR-024.

---

## Context

Milestone 10's own deliverable list never specified a charge/distribution step, and the M10
contributor audit review (Finding #1) confirmed the gap empirically: no code anywhere in
`internal/payment` ever constructs an `OwnerCharge` or `EscrowDeposit` event outside test
fixtures, so `mv_provider_escrow_balance` is structurally always 0 for every real provider —
without this engine, the payment system cannot pay anyone. `owner_escrow_events`'s own migration
comment describes `CHARGE` as "monthly storage deduction per active file (per-audit-pass
credits)," which reads as a design decision but was never ratified as one: no REQ or ARCH text
mandates it, and building against an unconfirmed inline comment risked shipping the wrong
incentive model in the first version of new, real-money-moving code.

ADR-012 already decided providers are paid for passing audits, not for GB stored — but that
decision is implemented entirely at the **release** stage, via `ComputeMonthlyRelease`'s
score-based multiplier (FR-049/050), which reads `mv_provider_scores` and shrinks a low-reliability
provider's payout independently of this decision. The open question this ADR resolves is
narrower and different: when a file's fixed monthly charge is credited as `DEPOSIT` events to
that file's current shard-holding providers, should that initial credit *also* be weighted by
each provider's audit-pass count for that file, or should it be split evenly and let the
already-existing release multiplier be the sole place reliability discounts apply.

Two design-council sessions examined this. The first considered a flat split (Option A) against
an audit-pass-weighted split (Option B) and a hybrid with a zero-audit fallback (Option C), and
converged on Option A via four independent arguments: reliability is already priced at release
time (double-counting it at credit time was judged redundant, not additive); production audit
cadence produces low per-file sample counts that make per-period weighting noisy; a hybrid's
fallback condition is itself a manipulable seam (a provider can force fallback by not answering
the first audit of a period); and a first version of new money-moving code carries meaningfully
less risk with simple integer division than with a cross-table proportional query. The second
session, prompted by a counter-proposal to weight distribution by audit passes while keeping
charge computation independent, surfaced a real attribution argument for weighting (a mid-period
repair reassignment means the provider holding a shard *right now* may not be who actually served
it for most of the period) but also surfaced a concrete new risk specific to weighting: the
`DEPOSIT` idempotency key `SHA-256(provider_id||file_id||billing_period)` carries no amount
component, so a charge run that executes even slightly before a billing period's audit data is
stable would lock in a wrong split permanently, with no retry path to correct it — turning an
operational sequencing requirement into a hard money-safety dependency. Weighed against that,
the project elected to ship the flat model and invest the saved complexity into scheduling
discipline for the charge job itself, rather than into weighted-distribution query logic.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| A — **Flat, split by shard count held (chosen)** | No sequencing dependency — can run at any time without waiting on audit data to close; O(1) per file; no float, minimal new surface area in a first version of money-moving code; does not duplicate the reliability signal ADR-012 already prices at release time | Attributes a full period's credit to whoever holds a shard at charge time, not who served it for the period (mid-period repair reassignment case) |
| B — Audit-pass-weighted | Matches the literal "per-audit-pass credits" schema comment; rewards actual verified availability, not just current possession | Requires the charge job to wait for the billing period's audit data to be stable before running — the fixed, amount-less idempotency key means a premature run locks in a wrong split with no correction path; low per-file audit sample sizes at production cadence make the weighting noisy; duplicates the reliability discount ADR-012 already applies at release time |
| C — Audit-pass-weighted with flat-split fallback on zero recorded passes | Handles the zero-audit edge case B leaves open | Strictly dominated by A: inherits all of B's sequencing risk and reliability double-counting, while its fallback condition is itself gameable — a provider can force the flat-split fallback for a file+period by simply not answering that period's first audit |

## Council Record (condensed)

*Session 1 (Option A vs. B vs. C):* Adversary — C's fallback is a fresh exploit surface, strictly
worse than A. Systems Theorist — weighting duplicates the reliability signal ADR-012 already
prices at release; a DEPOSIT should represent "holds this data now," not a second reliability
discount. Scale Advocate — B/C both need one batched query per billing run (not per-file), but
low audit sample sizes at production cadence make weighting noisy. Outsider — "per-audit-pass
credits" is an unconfirmed inline comment, not a ratified requirement. Implementer — A is the
lowest-risk first version; B/C need correct largest-remainder proportional math, meaningfully more
surface area. *Verdict: Option A.*

*Session 2 (re-examining after a charge-independent-of-distribution counter-proposal):* Adversary
— the amount-less idempotency key turns weighting's sequencing requirement into a permanent-drift
risk if the charge job ever runs early. Systems Theorist — partially retracted the pure
double-counting argument: mid-period repair reassignment is a real attribution problem flat split
gets wrong, independent of reliability-pricing. Scale Advocate — A has no sequencing dependency at
all; B/C's real production cost is operational discipline (must wait for period close), not query
cost. Outsider — flagged that "charge" must stay independently rate-based, not derived from
distribution, or the pricing-estimate and reserved-balance subsystems break. Implementer —
solvency (charge = sum of distributed amounts) is a construction property or either option, not an
option choice; proposed a runtime `sum(deposits) == charge` assertion regardless of verdict.
*Verdict (Council): Option C′ (weighted, charge kept independent). Final call (Aryan): Option A,
with the complexity budget redirected into charge-job scheduling discipline instead.*

## Decision

The charge/distribution engine (`internal/payment/charge.go`) charges each `ACTIVE` file's owner
its monthly storage cost, computed independently of any distribution logic, and splits that exact
amount evenly across the file's currently-`ACTIVE` `chunk_assignments`, weighted by shard **count**
held (a provider holding 2 of a file's active shards receives twice the credit of a provider
holding 1), not by provider identity alone.

1. **Cost.** `fileMonthlyCostPaise(sizeBytes, profile)` — integer-only reimplementation of
   `internal/api`'s `fileMonthlyCostPaiseForBytes` (round-half-up:
   `(sizeBytes*profile.StorageRatePaisePerGBPerMonth + bytesPerGB/2) / bytesPerGB`), duplicated
   rather than imported because `internal/payment` cannot import `internal/api` (IC §9) and cannot
   use `float64` anywhere (IC §11 forbidden-pattern list) — the existing function uses `float64`,
   acceptable for a user-facing estimate, not acceptable for a value actually debited.
2. **Charge.** One `OwnerCharge` event per file per billing period, idempotency key
   `SHA-256(owner_id||file_id||billing_period)` (`owner_escrow_events` migration comment, already
   specified). `billingPeriod` is a `"YYYY-MM"` string, matching `shouldRunRelease`'s existing
   convention (ADR pending for Finding #6's fix — same file).
3. **Distribution.** For the same file, one `EscrowDeposit` event per distinct provider currently
   holding at least one `ACTIVE` `chunk_assignment` for that file (across all its segments),
   idempotency key `SHA-256(provider_id||file_id||billing_period)`. Amount per provider =
   `charge * providerShardCount / totalActiveShardCount`, computed via the largest-remainder
   method so `sum(deposits) == charge` exactly on every run — the base share is
   `floor(charge * providerShardCount / totalActiveShardCount)` for every provider, and the
   leftover `charge - sum(baseShares)` paise are distributed one each, in `provider_id` ascending
   order for determinism across retries, to the providers with the largest fractional remainder.
4. **Scheduling discipline.** The charge job's own calendar loop reuses the exact pattern Finding
   #6 established for `runReleaseOnCalendarDate`/`shouldRunRelease`: a pure
   `shouldRunCharge(now time.Time, lastRun string) (run bool, newLastRun string)` function keyed on
   `now.Format("2006-01")`, never a bare month number — closing off the same annual-rollover class
   of bug before it can recur in a second scheduler. Unlike the release loop, the charge loop has
   no audit-period-closed precondition to wait for (Option A's whole point per the council):
   it may run on any fixed day of the month against "now."
5. **Scope of "current holder."** Only `status = 'ACTIVE'` `chunk_assignments` count — the same
   filter `active_chunk_assignments` (the view already granted to `vyomanaut_app`) uses. A
   `REPAIRING` assignment belongs to a holder already on its way out; it is not credited further
   by this engine.

## Consequences

**Positive:**

- Charge computation has no sequencing dependency on audit-period closure — it can run
  deterministically on a fixed calendar cadence, the way release computation cannot.
- No new cross-table proportional query in the first version of a new money-moving code path;
  `internal/payment` stays float-free throughout.
- Does not duplicate the reliability discount ADR-012 already applies at release time via the
  score multiplier — the two payment stages price two different things (current possession vs.
  proven reliability) rather than the same thing twice.
- `sum(deposits) == charge` is a construction invariant of the largest-remainder method, not a
  hope; every fix in this milestone's other corrections (idempotency, guard-before-insert)
  applies identically here.

**Negative / trade-offs:**

- A provider who takes over a shard via mid-period repair reassignment receives that period's
  full per-shard credit even though a different provider served most of the period — flat split
  does not track *when* within a period a shard changed hands, only who holds it at charge time.
- The reliability discount that does exist is applied once, at release, on the provider's
  *aggregate* score across all files — a provider that is unreliable specifically on one file but
  fine elsewhere is not penalised at the per-file level by this engine; only the release-stage
  aggregate multiplier catches it, and only in aggregate.

**Open constraints:**

- If audit-pass weighting is revisited later, it needs its own ADR, not a silent change to this
  one — the council's Q1 answer (Round 2) makes the operational precondition (billing period must
  be closed before the charge job may run) a hard requirement for that future decision, not an
  implementation detail.
- The largest-remainder determinism (provider_id ascending tie-break) must never change without a
  migration note — a retried charge run that redistributed the remainder paise differently would
  silently violate the idempotency guarantee this design otherwise provides.
- `shouldRunCharge`'s day-of-month constant must stay a named constant (never hardcoded inline),
  matching the existing discipline enforced everywhere else in this file's sibling,
  `runReleaseOnCalendarDate`.

## References

- [ADR-012 — Pay Providers Per Audit Passed, Not Per GB Stored](ADR-012-payment-basis.md): the
  reliability-pricing mechanism this ADR deliberately does not duplicate at charge time
- [ADR-016 — Payment DB Schema: PN-Counter CRDT with Idempotency Key](ADR-016-payment-db-schema.md):
  idempotency key discipline this ADR's charge/deposit keys follow
- [ADR-024 — Economic Mechanism: Deterministic Fiat Escrow with Graduated Penalty](ADR-024-economic-mechanism.md) §4–5:
  confirms seizure-on-departure is already all-or-nothing by tenure, independent of this decision
- `data-model.md` §4.3 (`files`), §4.4 (`segments`), §4.5 (`chunk_assignments`): shard-holder
  query shape — a file's shard-holders are the union across all its segments' `ACTIVE`
  `chunk_assignments`, not a single per-file set
- `owner_escrow_events` migration comment (`CHARGE` idempotency key formula, already specified
  before this ADR)
- M10 contributor audit review, Finding #1 (the scope gap this ADR closes) and Finding #8 (the
  `shouldRunRelease` pattern this ADR's scheduling discipline reuses)
