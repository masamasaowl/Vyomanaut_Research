# ADR-024 — Economic Mechanism: Deterministic Fiat Escrow with Graduated Penalty

**Status:** Accepted
**Topic:** #18 Economic Mechanism Design
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 5, 29, 31, 33, 35, 37, 40, 41

---

## Context

[ADR-011](./ADR-011-escrow-payments.md) fixed the payment instrument (Razorpay/UPI fiat escrow) and [ADR-012](./ADR-012-payment-basis.md) fixed the payment
unit (per audit passed). Both left open the economic mechanism parameters: how much is held in
escrow at any time, how long earnings are withheld, what graduated penalties apply before full
seizure, and how per-audit pricing is set. Without these parameters, the escrow service cannot
be implemented, providers cannot compute their expected earnings, and the microservice cannot
enforce SLA violations.

Four papers now bound the design space completely:

- Storj ([Paper 05](../research/paper-05-storj.md) ) provides the closest production precedent to Vyomanaut's held-earnings model: earnings held for nine months, reclaimed on premature departure to cover repair costs (Section 4.16). Filecoin uses pre-committed collateral (stake before earning); Storj holds accumulated earnings. Vyomanaut follows the Storj model — held-earnings, not pre-committed stake — because it matches the India-first provider demographic where pre-commitment creates an unacceptable capital barrier.
- Filecoin ([Paper 29](../research/paper-29-filecoin-whitepaper.md)) provides the graduated penalty template: score decrement per missed proof,
  partial penalty at Δfault/2, full cancellation at Δfault.
- PeerTrust ([Paper 31](../research/paper-31-peertrust.md)) provides the adaptive dual-window scoring pattern: reputation is hard to
  build, easy to lose; the cost of rebuilding must exceed the gain from milking.
- Ihle et al. ([Paper 33](../research/paper-33-ihle-incentive-mechanisms.md)) provides the decision tree validation: deterministic pricing + non-transferable accumulated earnings = the correct incentive type for contracted cold storage.

The 72-hour departure threshold ([ADR-006](./ADR-006-polling-interval.md), [ADR-007](./ADR-007-provider-exit-states.md)) and the three-window reliability score
([ADR-008](./ADR-008-reliability-scoring.md)) are fixed inputs. This ADR specifies everything layered on top of them.

---

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Pre-committed collateral (stake before earning, Filecoin model) | Deters low-MTTF entry; providers have skin in game before receiving any data | Capital barrier eliminates legitimate Indian home desktop providers who will not pre-commit ₹; contradicts India-first target demographic |
| Fully delayed payment (hold 100% of earnings until contract end, Gramaglia model) | Maximum deterrent to exit at any point | Providers have zero cash flow during the contract period; economically infeasible for participants who depend on monthly income |
| **Held-earnings escrow with sliding release window (chosen)** | No capital entry barrier; providers earn and partially receive payments monthly; escrow buffer scales with chunk count; proportional penalty preserves partial fairness | Requires microservice to track per-provider escrow state; payment gateway must support partial hold-and-release |

---

## Decision

### 1. Pricing model

**Per-audit payout rate:** The base payout per audit pass is:

```
payout_per_audit_pass = (storage_rate_paise_per_GB_per_month × chunk_size_GB) / audits_per_month
```

Where:
- `chunk_size_GB` = 256 KB / (1024 × 1024) ≈ 0.000244 GB
- `audits_per_month` = 30 (one audit challenge per chunk per day, 30-day month)
- `storage_rate_paise_per_GB_per_month` = product decision, set at contract creation

The storage rate is **deterministic and fixed at contract creation**. It does not fluctuate with
network demand. Data owners and providers both know their exact expected costs and earnings
before any data is transferred. This is the deterministic pricing pattern validated by [Paper 33](../research/paper-33-ihle-incentive-mechanisms.md).

Per-segment pricing (Q05-8) is not adopted in V2. Metadata overhead (~44 bytes per chunk in
the WiscKey index, [ADR-023](./ADR-023-provider-storage-engine.md)) is negligible at the chunk sizes and volumes anticipated at launch.
Revisit if segment count grows to the point where metadata cost is measurable.

### 2. Escrow hold structure
> **Note:** The computation runs on the 23rd; the Razorpay release fires on the next business day after on_hold_until, targeting the first 3 business days of the following month.
>

At any given time, a provider's earned-but-not-yet-released escrow balance is bounded by a
**rolling hold window of 30 days**. Earnings older than 30 days are released automatically on
the first of each month, provided the provider's 30-day reliability score is above the release
threshold (see §4).


Storj [Paper-05](../research/paper-05-storj.md) precedent: Storj holds earnings for nine months (Section 4.16). Vyomanaut's 30-day rolling window is shorter, calibrated to provider cash-flow expectations for Indian home desktop providers. The seizure mechanism (all held earnings reclaimed on silent departure to fund repair) is identical in principle.

```
escrow_held_at_time_T = sum of audit-pass earnings in [T-30d, T]
escrow_releasable    = sum of audit-pass earnings in [T-60d, T-30d]
                       × release_multiplier(score_30d)
```

The maximum amount a provider has at risk at any time is approximately 30 days of earnings.
For a provider storing 1,000 chunks at the minimum viable storage rate, this is a predictable,
bounded figure disclosed at onboarding — not a pre-committed stake.

Razorpay Route's on_hold_until releases on the next business day after the timestamp, not on the timestamp date itself. The release window should read "within the first 3 business days of each month" rather than "on the first of each month." The microservice must set on_hold_until to midnight on the last working day of the current month (accounting for RBI bank holidays) so the release lands within the correct window.

**Non-transferability constraint:** The escrow balance is identity-bound to the registered
provider_id. Transfer of escrow balance to another provider_id is refused by the payment
microservice unconditionally. This enforcement is at the payment service level, not the database
level, making it auditable and enforceable even if the underlying Postgres row is visible.

### 3. Payout release multiplier

The release multiplier converts the 30-day reliability score into a proportional release factor:

```
score_30d = weighted_audit_pass_rate over trailing 30 days (from ADR-008)

release_multiplier(score) =
  1.00   if score ≥ 0.95   (full release — provider is reliable)
  0.75   if score ≥ 0.80   (partial release — minor degradation detected)
  0.50   if score ≥ 0.65   (partial release — significant degradation)
  0.00   if score < 0.65   (hold in full — earnings retained pending investigation)
```

The withheld portion on a partial release is **not seized**. It is rolled forward into the next
month's escrow window. It is only seized if the provider subsequently exits silently (§5).

The thresholds (0.95, 0.80, 0.65) are starting values. They must be tuned empirically after
V2 launch using provider telemetry. The principle — that the release multiplier makes it costly
to degrade below a reliability threshold without fully penalising transient hardware failures —
is the PeerTrust adaptive window pattern ([Paper 31](../research/paper-31-peertrust.md), Section 4.3) applied to payment release.

Implementation note: Route does not support percentage-based transfers. The microservice must compute escrow_held × release_multiplier as an integer paise amount and pass that to the Transfer API. All arithmetic must use integer paise (consistent with [ADR-016's](./ADR-016-payment-db-schema.md) amount_paise BIGINT column).

### 4. Graduated penalty timeline

Filecoin's Manage.RepairOrders three-case structure maps to Vyomanaut as follows:

| Time since last contact | State | Action |
|---|---|---|
| 0–24 h | Temporary absence | Score decrement per polling cycle ([ADR-008](./ADR-008-reliability-scoring.md)); no payment action |
| 24–48 h | Degraded | Score continues to fall; release multiplier may drop; partial hold begins on next release date |
| 48–72 h | Warning zone | If 48 h threshold is reached and provider has not responded, queue a repair pre-assessment: identify candidate replacement providers but do not initiate data transfer yet |
| ≥ 72 h | Silent departure | Trigger repair ([ADR-004](./ADR-004-repair-protocol.md)); seize all held escrow (the rolling 30-day window); mark provider as departed; sign provider out of network ([ADR-007](./ADR-007-provider-exit-states.md)) |
| Any time | Announced departure | Trigger repair immediately; release held escrow proportional to completion of the current 30-day window; sign provider out of network |

**Partial hold trigger:** If the 7-day reliability score drops more than 0.20 below the 30-day
score (PeerTrust dual-window signal, Q31-1), a partial hold flag is set immediately. The
next scheduled monthly release uses the degraded release multiplier even if the departure
threshold has not been crossed. This catches providers who are milking their reputation before
a planned silent departure.

```
dual_window_flag = (score_30d - score_7d) > 0.20
if dual_window_flag: next_release_multiplier = min(release_multiplier(score_30d),
                                                    release_multiplier(score_7d))
```

### 5. Escrow seizure mechanics

On silent departure (t ≥ 72 h with no contact):
1. Payment microservice freezes the provider's escrow account.
2. All earnings in the 30-day rolling window are seized (transferred to the repair reserve fund).
3. The repair reserve fund is used to subsidise the cost of onboarding replacement providers
   for the departed provider's chunks.
4. A seizure receipt is generated and appended to the audit log (I-confluent INSERT — [ADR-013](./ADR-013-consistency-model.md)).
5. Provider is soft-deleted in the provider table (ADR-013: physical deletion prohibited).

The repair cost covered by seized escrow is bounded: Qpeek ≈ 793 GB for N=1000 ([ADR-004](./ADR-004-repair-protocol.md)).
At any provider MTTF above 180 days, the seized escrow from a departing provider will exceed
the repair bandwidth cost. If it does not (very low-MTTF provider), the deficit is covered by the
repair reserve fund, which accumulates from partial-release withheld amounts over time.

### 6. Vetting period economics

During the 4–6 month vetting period ([ADR-005](./ADR-005-peer-selection.md)), providers receive full audit-pass payouts but
under the following modified hold structure:

- Hold window: 60 days (double the post-vetting window)
- Release multiplier: capped at 0.50 until vetting is complete

This means a provider in the vetting period can earn but can access at most 50% of any month's
earnings until they pass the vetting threshold. The extended hold mirrors the entry cost pattern
identified in [Paper 33](../research/paper-33-ihle-incentive-mechanisms.md) (Section 4.2.3): an economic barrier to rapid Sybil registration and
immediate withdrawal.

After vetting is complete, the hold window reverts to 30 days and the release multiplier ceiling
is removed.

### 7. Escrow floor for repair reserve

The repair reserve fund maintains a minimum floor of:

```
repair_reserve_floor = N_active_providers × lf × (r - r0) × cost_per_GB_transfer
```

Where N_active_providers is the current count, lf = 256 KB, r - r0 = 32 (repair bandwidth
gap), and cost_per_GB_transfer is the marginal cost of P2P transfer (negligible in the V2
model where providers bear their own bandwidth cost). In V2, the repair reserve floor is
primarily a bookkeeping constraint, not a hard capital requirement, since repair bandwidth
cost is borne by the receiving replacement provider, not by the microservice.

**Theorem 1 conditions ([Paper 37](../research/paper-37-shelby-incentive-compatibility.md), Crystal et al.):** Vyomanaut's economic design satisfies the three formal conditions for honest storage equilibrium. At V2's daily full-audit frequency (pau ≈ 1 per chunk per day):

- Condition (i) — slashing discourages false reporting — trivially satisfied since (1−pau)/pau ≈ 0;
- Condition (ii) — audit reward ≥ audit cost — satisfied since disk read + hash computation is negligible;
- Condition (iii) — storage reward ≥ storage cost — the participation constraint set by the storage rate.

If audit frequency is reduced in V3 (probabilistic sampling, pau << 1), Condition (i) must be verified against the new pau value before reducing.

---

## Consequences

**Positive:**
- No capital barrier to entry — any Indian provider with a UPI-linked bank account can join
  without pre-committing funds
- Monthly partial releases give providers predictable cash flow while maintaining a deterrent
  to exit (30-day rolling window always at risk)
- Graduated penalty distinguishes transient hardware failures from intentional departure —
  providers are not punished harshly for a weekend absence
- Dual-window score trigger catches providers who begin degrading before the 72h threshold,
  applying the partial hold before the exit is confirmed
- Vetting period hold structure provides an entry cost without requiring a financial stake

**Negative / trade-offs:**
- Payment microservice must track per-provider escrow state, release schedules, and seizure
  events — this is operationally more complex than a simple per-audit transfer
- Release multiplier thresholds (0.95, 0.80, 0.65) are starting values that require empirical
  calibration after launch; wrong thresholds create perverse incentives (too lenient: no
  deterrence; too strict: punishes legitimate hardware failures)
- The 30-day hold means a provider who exits cleanly on day 1 leaves 30 days of earnings in
  escrow until the next release date — this must be disclosed clearly at onboarding to avoid
  provider dissatisfaction
- Razorpay escrow API must support partial hold-and-release semantics; if it does not, a
  custom escrow ledger within the microservice is required (and Razorpay is used only for
  the final fiat transfer)

**Open constraints:**
- Release multiplier thresholds must be tuned empirically at V2 beta launch
- Razorpay Escrow API documentation (reading list remaining item #6) must be read before
  implementation to confirm whether native partial-release is supported or a custom ledger
  is required
- Storage rate (paise per GB per month) is a product pricing decision not made in this ADR;
  it must be set before any contract can be signed
- Held-earnings percentage for the vetting period (50% cap) is a starting value; if provider
  retention during vetting is low, consider raising the cap to 75%

---

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): held-earnings vetting model (nine-month hold, reclaim on premature departure) — the closest production precedent to Vyomanaut's escrow structure
- [Paper 29 — Filecoin](../research/paper-29-filecoin-whitepaper.md): graduated penalty structure (Manage.RepairOrders); collateral proportional to storage; repair re-introduction mechanics
- [Paper 31 — PeerTrust](../research/paper-31-peertrust.md): adaptive dual-window scoring; cost of rebuilding reputation must exceed gain from milking; partial hold trigger design
- [Paper 33 — Ihle et al.](../research/paper-33-ihle-incentive-mechanisms.md): decision tree selecting deterministic + non-transferable accumulated incentive; deterministic vs. auction pricing trade-offs; entry cost as Sybil deterrent; Gramaglia long-term P2P storage as closest architectural precedent
- [Paper 35 — Razorpay API Docs](../research/paper-35-razorpay-upi-docs.md): on_hold_until releases on next business day (not exact date); paise-integer arithmetic mandatory; Route not Escrow+; UPI Intent replaces deprecated Collect flow
- [Paper 37 — SHELBY](../research/paper-37-shelby-incentive-compatibility.md): Theorem 1 three conditions for honest equilibrium; at V2 daily audit frequency, Condition (i) is trivially satisfied; threshold must be checked if audit frequency is reduced
- [Paper 40 — Buragohain et al.](../research/paper-40-buragohain-p2p-incentives.md): Nash conditions satisfied at any positive storage rate and N > 1; stable high-contribution equilibrium self-reinforces as provider count grows
- [Paper 41 — Vakilinia et al.](../research/paper-41-vakilinia-incentive-compatible-dsn.md): bilateral game payoff conditions already satisfied by escrow seizure mechanism; confirms subgame-perfect equilibrium is {Store, No Challenge}
- [ADR-006](ADR-006-polling-interval.md): 72-hour departure threshold; 24-hour polling interval
- [ADR-007](ADR-007-provider-exit-states.md): four exit states; repair trigger conditions; escrow seizure trigger
- [ADR-008](ADR-008-reliability-scoring.md): three-window rolling score; 24h/7d/30d windows; score decrement mechanics
- [ADR-011](ADR-011-escrow-payments.md): Razorpay/UPI fiat escrow; PaymentProvider interface
- [ADR-012](ADR-012-payment-basis.md): payment per audit passed; decoupling from P2P transfer layer
- [ADR-013](ADR-013-consistency-model.md): escrow debit and seizure are non-I-confluent; single payment service required
- [ADR-016](ADR-016-payment-db-schema.md): PN-counter CRDT escrow_events table; idempotency key on releases
- [ADR-005](ADR-005-peer-selection.md): 4–6 month vetting period; vetting period economics constraint