# ADR-066 — Prometheus metric unit grammar, with an explicit dimensionless class

**Status:** Accepted
**Topic:** #17 Observability *(new topic)*
**Supersedes:** — *(amends NFR-046; retires NFR-042 per D-06)*
**Source:** This review; live registry audit of `internal/metrics/*.go`; the `[Decision — OBS.2.1]`
note at `internal/metrics/daemon.go:7–15`

## Context

NFR-046 mandates `vyomanaut_{subsystem}_{name}_{unit}` with unit ∈ `{_total, _seconds, _bytes}`.
Six of fifteen registered metrics violate it (D-08). Two violate it knowingly — the implementer
documented the conflict at the call site and chose fidelity to the frozen OBS.2.1 table over a
silent rename, which was the right call under the project's own "flag, don't patch" rule.

The requirement also contradicts itself: its own illustrative examples include
`vyomanaut_repair_queue_depth` and `vyomanaut_provider_score`, both unitless gauges, under a grammar
that admits no unitless form.

Timing is the whole point. NFR-046 defines a metric rename as a breaking change once Grafana
dashboards reference it. Today the dashboards are internal and a rename costs one PR. After
external providers or an institutional deployment exist, it costs a coordinated migration. **M18 is
the last cheap moment**, which is why this belongs here rather than in a later cleanup.

## Updates

Q-M18-3 resolved: the dimensionless class has two members — count (queue depth, replica count) and state (boolean gauges such as vyomanaut_daemon_ram_constrained), each declared in DimensionlessNames with its kind. Second open constraint resolved: NFR-046's "subsystem matches the internal/ package name" clause softens to "matches an internal/ package name or a declared pseudo-subsystem," with db and daemon declared.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Rename both histograms to `_seconds`; add a dimensionless class — chosen** | Prometheus base-unit convention is near-universal; `histogram_quantile` output becomes directly comparable with `vyomanaut_db_read_latency_seconds`; Grafana formats seconds→ms for humans at display time, so nothing human-facing degrades | Two renames now; the FR-029 CLI status view must divide by 1000 at render |
| **Formalise a local-only exempt namespace** (`vyomanaut_local_*`, non-scraped) | Zero renames; honours the OBS.2.1 reasoning that these feed a loopback CLI, not Prometheus | Creates two grammars and a boundary that will be argued about at every future metric; and the boundary is wrong — the provider daemon's `/metrics` is exactly what a self-hosting institutional operator would scrape |
| **Drop the unit-suffix rule; keep only the prefix rule** | Simplest; makes the tree conformant immediately | Discards the only property that makes a metric name self-describing; ADR-066 would be deleting a requirement to avoid fixing six metrics |

## Decision

1. **Unit vocabulary:** `_total` (counters), `_seconds` (all duration histograms and summaries),
   `_bytes` (size gauges and histograms), `_ratio` (0–1 gauges).
2. **Dimensionless class — explicit and closed.** A metric measuring a *count of things currently
   in a state* (queue depth, replica count, boolean state) or a *unitless score* takes **no unit
   suffix**, and must be declared in an allow-list constant `internal/metrics.DimensionlessNames`.
   Being on that list is a deliberate act, not a default. This resolves NFR-046's self-contradiction
   by making its own examples legal.
3. **`_milliseconds` is prohibited.** `DaemonAuditResponseLatencyMilliseconds` and
   `DaemonVlogAppendLatencyMilliseconds` are renamed to `..._seconds` and record seconds. The CLI
   status view (FR-029) formats for humans at the presentation layer, where formatting belongs.
4. **CI check 16 is extended** from orphan-name detection to full grammar conformance: prefix,
   subsystem allow-list, unit vocabulary, and dimensionless allow-list membership.
5. **NFR-042 retired** (D-06); NFR-046 rewritten to carry the grammar above.

## Consequences

The registry becomes conformant and the conformance becomes machine-checked rather than
review-checked. Future metrics fail CI at the moment of definition. Cost: two renames and one
render-layer divide, both now rather than after an external dashboard exists.

**Open constraints:**

- `vyomanaut_daemon_ram_constrained` is a boolean gauge. Prometheus convention would prefer
  `..._ram_constrained` as a `0|1` gauge (which it is) — but "boolean state" and "count of things"
  are different enough that the dimensionless class arguably needs two members. Not blocking;
  flagged so the allow-list comment records the reasoning. Q-M18-3.
- The subsystem allow-list `{audit, scoring, repair, payment, cluster, storage, daemon, db}` is
  inherited from OBS.4.1 and contains `db` and `daemon`, neither of which is an `internal/` package
  name — so it already violates NFR-046's *"subsystem matches the `internal/` package name"* clause.
  Either the clause softens to "subsystem or documented pseudo-subsystem" or the two are renamed.
  Recorded, not decided here.

---
