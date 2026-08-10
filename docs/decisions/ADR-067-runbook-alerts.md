# ADR-064 — Every runbook has an alert; every alert has a runbook

**Status:** Proposed — blocked on ADR-063 (alert expressions must reference final metric names)
**Topic:** #17 Observability
**Supersedes:** — *(extends NFR-027; adds a ninth runbook — see ADR-065)*
**Source:** This review; `MVP §8.5`; `IC §10`; live audit of `deployments/grafana/alerts.yaml`

## Context

Two documents state that Grafana alerts link to runbooks by name, and that this coupling is why the
eight filenames are frozen. The coupling does not exist: no rule in `alerts.yaml` carries a
`runbook_url` annotation. Meanwhile five of eight runbooks have no alert that could ever open them,
and Session 18.1.1 requires each to name one.

This is not a documentation defect. A runbook nobody is paged into is a document, not a control.
The five without alerts are exactly the ones covering third-party and scheduled failure modes —
Postgres, the secrets manager, Razorpay, the RBI holiday table, secret rotation — which is to say,
the failures an on-call engineer is *least* likely to diagnose from first principles at 3 a.m.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Weaken Session 18.1.1 to "trigger conditions, alert name if one exists"** | Zero engineering; M18 closes | Ratifies a launch certificate that documents five uninstrumented failure modes as covered |
| **Define the five missing alerts; leave linkage as convention** | Closes the detection gap | Convention is what produced the current state; nothing prevents drift |
| **Define the missing alerts and make the bijection CI-enforced — chosen** | Detection gap closed *and* held closed; the frozen-filename rule finally does the job it was written for | Five new alerts need backing metrics, two of which do not exist yet |

## Decision

**1. Five new alerts**, added to `deployments/grafana/alerts.yaml` and to `architecture.md §23`'s
alert table:

| Alert | Condition | Severity | Runbook | Backing metric |
| --- | --- | --- | --- | --- |
| `PostgresPrimaryUnreachable` | `up{job="postgres"} == 0 for 2m` | critical | `postgres-failover.md` | exporter `up` — exists |
| `SecretsManagerUnavailable` | `increase(vyomanaut_cluster_secret_fetch_failures_total[15m]) > 0` | critical | `secrets-manager-outage.md` | **new** — IC §8 already defines `ErrSecretManagerUnavailable`/`ErrSecretExpired`; this counts them |
| `RazorpayAPIErrorRateHigh` | `rate(vyomanaut_payment_razorpay_errors_total[15m]) / rate(vyomanaut_payment_razorpay_requests_total[15m]) > 0.05` | critical | `razorpay-api-outage.md` | **new** |
| `RBIHolidayTableStale` | `vyomanaut_payment_rbi_holiday_table_year < year(now())+1` | warning | `rbi-holiday-table-update.md` | **new** (dimensionless, ADR-063) |
| `AuditSecretRotationOverdue` | `time() - vyomanaut_cluster_audit_secret_rotated_timestamp_seconds > 90d` | warning | `audit-secret-rotation.md` | **new** |

`RBIHolidayTableStale` is the one I would most defend on its own merits: NFR-031's annual December
update is currently guarded by a runbook and a human calendar. A gauge carrying the table's newest
year turns a silent annual omission — which would misprice every `on_hold_until` date until someone
noticed — into a warning that fires with eleven months of lead time.

**2. `runbook_url` becomes mandatory** on every rule, pointing at the repository path of an existing
runbook file.

**3. Bijection is CI-enforced** (new check 17, `TestAlertRunbookBijection`): every file in
`runbooks/` is referenced by ≥1 alert's `runbook_url`; every `runbook_url` resolves to a file that
exists. Both directions.

## Consequences

Nine runbooks (eight plus ADR-065's), nine-plus alerts, bidirectionally verified. Three new
counters and two new gauges enter the NFR-025 frozen set and must be added to it in the same PR.

**Open constraints:**

- `PostgresPrimaryUnreachable` depends on a `postgres_exporter` in the deployment, which
  `deployments/production/` does not currently specify. Blocking for the alert, not for this ADR.
- The 90-day rotation interval in `AuditSecretRotationOverdue` is a starting value. IC §8 specifies
  the 24-hour *overlap window* but no rotation *cadence*. Q-M18-4.

---
