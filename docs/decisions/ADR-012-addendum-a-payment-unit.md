# ADR-012 Addendum A — The Audit Is a Gate, Not a Meter

**Status:** Accepted
**Track:** LTS
**Topic:** #7 Payment Model
**Amends:** ADR-012 (its incentive argument stands; its statement of the *unit of account* does not)
**Closes:** F-03 · **Confirms:** ADR-061 · **Unblocks:** ADR-060
**Research source:** Design Council session 3, August 2026; live verification of
`internal/payment/{charge,release}.go`

---

## Why this addendum exists

ADR-012 states: *"Providers are paid for passing audits, not for successful retrievals or GB
stored."* The implemented payment path does something else, and has since M10:

| Stage | Code | Behaviour |
| --- | --- | --- |
| Charge | `internal/payment/charge.go:67` | `fileMonthlyCostPaise = (sizeBytes × StorageRatePaisePerGBPerMonth + ½GB) / GB` — **per GB, per month** |
| Distribute | `splitByLargestRemainder` (ADR-061) | flat across current shard-holders by shard **count** |
| Release | `internal/payment/release.go` | escrow balance × a tier multiplier keyed on `mv_provider_scores` — a pass **rate** (10000/7500/5000/0 bp at 0.95/0.80/0.65) |

**Nothing anywhere multiplies by a count of audits passed.** The implemented model is *charge per
GB-month, gate release on audit-pass rate* — which is one of the two clean answers ADR-060 offers
for F-03. F-03 was blocking ADR-060's approval because *"the schema follows the payment unit."*

## The ruling

**The payment unit is per-GB-per-period. The audit is a gate, not a meter.** This is what the code
does; the ruling makes it deliberate rather than accidental.

Five independent arguments reached it in council, which is worth recording because convergence is
usually a warning sign and here it survived an explicit check:

1. **Farming vector.** Sampling under ADR-060 is 1% *per file*. Under per-audit-count income, a
   provider holding 1,000 tiny files is audited 1,000 times a day while a provider holding one large
   file with the same bytes is audited once. Income would scale with file *count* — a quantity the
   provider can solicit. Per-GB-month has no such vector.
2. **Layering.** ADR-012 conflates the *incentive* ("income should depend on continuous proven
   presence") with the *unit of account* ("income is denominated in audits"). The implemented path
   preserves the first and rejects the second. They are separable.
3. **Row count.** ADR-060's entire achievement is moving from O(chunks) to O(files). Denominating
   payment in audits hands that back.
4. **Predictability.** ADR-060 notes two providers storing identical data can earn different amounts
   under sampling. The provider-facing promise is predictable income from idle disk.
5. **Cost.** `charge.go` and `release.go` are built, integer-only and idempotent. Changing the unit
   rewrites both plus the receipt schema.

## Amendments to ADR-012

### 1. Distinguish per-GB **ingress** from per-GB **stored-per-period**

ADR-012's rejection of "per GB" cites Storj's delete-and-restore attack — nodes deleting data to be
paid for re-storing it. **That attack applies to per-GB ingress.** The code implements per-GB
*stored-per-period*, which has no such attack because nothing is paid for receiving data. The
rejection was correct and the label was over-broad. ADR-012's "Why not per GB ingress" section
stands unchanged; its headline and its "not GB stored" phrasing are corrected.

### 2. The incentive argument survives intact

Every reason ADR-012 gives for coupling income to proven presence — high-MTTF incentive, payment
decoupled from the P2P transfer layer, no credit liability accruing during a microservice outage —
holds exactly as written under a rate-based gate. Only the denomination changes.

## Effect on ADR-061 (F-LTS-04)

**ADR-061 is confirmed; no change.** ADR-060 does weaken one of ADR-061's four arguments for the flat
split — the claim that per-file audit sample counts are too low to weight on, which ADR-060's
per-`(provider, file, day)` receipt raises by roughly 287×. The other three are untouched:
reliability is already priced at release, Option C's fallback condition is gameable, and the DEPOSIT
idempotency key `SHA-256(provider_id‖file_id‖billing_period)` carries **no amount component**, so any
future weighted split makes a premature charge run permanently wrong with no correction path.

Recorded explicitly: **the premise changed and the verdict did not.** That third point is also a
standing constraint — weighting cannot be revisited without changing the idempotency key first,
which is a migration rather than a tweak.

## Residuals neither option fixes

Recorded against ADR-024, not closed here:

- **Targeted defection is under-punished.** Flat split plus an *aggregate* score multiplier means a
  provider deleting one owner's shards while serving everyone else's is penalised only in aggregate.
  → `reading-list.md` Domain V, R-70.
- **The operator now carries sampling risk.** Predictable provider income means the variance lands
  somewhere, and it lands on the operator. Uncosted.

## Consequences

F-03 closes, unblocking ADR-060's approval. No code change. ADR-061 confirmed and gains its missing
`Track:` line. Domain J's R-63 (time-weighted attribution) is downgraded to conditional-on-a-count:
its trigger was ADR-061's mid-period handover problem, and under ADR-076 repair becomes rare, so
handovers become rarer still — **count the handover fraction before funding a search; under ~1% it
closes unstarted.**

## References

- ADR-012 — amended here
- ADR-060 — F-03 was its blocking precondition
- ADR-061 — confirmed; F-LTS-04 resolves as no change
- ADR-024 — the two residuals land here
- ADR-076 — makes repair rare, which is what downgrades R-63
