# ADR-068 — Relay capacity telemetry and pre-emptive provisioning

**Status:** Accepted
**Topic:** #6 Network Layer / #17 Observability
**Supersedes:** — *(instruments `architecture.md §27.5`, §27.7, §27.8; closes the telemetry half of Q20-1)*
**Source:** This review; `architecture.md §27.5/§27.7/§27.8`; `IC §4.3`; NFR-006

## Context

`architecture.md §27.7` names its own first binding ceiling and then says how not to detect it:

> *"The most likely first constraint in the V2 growth trajectory is relay slot exhaustion. Relay
> infrastructure is cheap to add incrementally but must be provisioned ahead of the trigger — not in
> response to TIMEOUT alerts."*

There is no relay capacity metric. There is no relay capacity alert. The `relay-node-replacement.md`
runbook covers a relay *failing*, not relays *filling*. The only signal that exists is
`AuditTimeoutRateHigh` — the reactive signal the paragraph above explicitly rules out.

The numbers make this urgent rather than theoretical. §27.5:

```
Required slots  = N × relay_fraction × 1.5
Available slots = relay_nodes × 128
Provision when  required > 0.80 × available
 
At 3 relay nodes: available = 3 × 128 = 384 slots
45% CGNAT (Indian pessimistic): binds at N ≈ 570 providers
30% CGNAT (global baseline):    binds at N ≈ 850 providers
Stated operational rule: provision the 4th node before N = 400
```

Compare this against every other ceiling in §27.7. Postgres INSERT binds at ~100,000 providers.
Per-provider bandwidth binds at ~70 GB. Relay binds at **570** — and the mitigation requires
procuring, deploying and warming a new node, which is days of lead time, not minutes. It is
simultaneously the nearest ceiling and the slowest to remediate, and it is the only one with no
instrumentation at all.

There is a second payoff. §27.5 says *"Q20-1 telemetry at private beta resolves the actual CGNAT
fraction"* — the 30%-vs-45% assumption that swings the binding point by 280 providers. Nothing
currently collects that telemetry either. The same metric closes both.

## Updates

 Q-M18-5 resolved: relay nodes do not refuse reservations below hard exhaustion — rejecting a reservation rejects a provider from the network, and the 70% warning plus 85% critical give sufficient lead time. slots_total reads the configured limit at runtime, never the assumed 128. Now additionally gated on M19 Session 19.1.3, which is where Circuit Relay v2 reservations first exist.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — infer from TIMEOUT rate** | Nothing to build | Lagging by construction; a TIMEOUT spike is indistinguishable from provider absence, relay failure, or capacity exhaustion — the three things needing three different responses. Explicitly rejected by §27.7 |
| **Periodic manual capacity audit against the §27.5 table** | No code; matches how §27.8's scale milestones are already framed | Depends on someone running it on a cadence nobody has specified; fails exactly when the network is growing fastest |
| **Relay-side reservation gauges + utilisation alert + capacity runbook — chosen** | Leading indicator; instruments the architecture's own stated rule; closes Q20-1's telemetry as a side effect | Requires relay nodes to expose `/metrics`, which `internal/p2p`'s relay mode does not currently do |

## Decision

**1. Three new metrics.** On each relay node (`cmd/microservice --relay-mode`):

```
vyomanaut_relay_reservation_slots_used     gauge, dimensionless (ADR-066)
vyomanaut_relay_reservation_slots_total    gauge, dimensionless
vyomanaut_relay_reservations_rejected_total  counter
```

And on the microservice, closing Q20-1:

```
vyomanaut_cluster_relay_dependent_providers  gauge, dimensionless
```

incremented per provider whose heartbeat multiaddr (IC §3.1) contains a `/p2p-circuit` component —
which is the CGNAT fraction, measured rather than assumed, at zero additional protocol cost.

**2. Alert at 70%, not 80%.** §27.5's 80% is the *provisioning* threshold. An alert must fire early
enough for a human to procure and deploy a node:

```
RelayCapacityHigh    (warning)  sum(slots_used) / sum(slots_total) > 0.70 for 1h
RelayCapacityCritical (critical) sum(slots_used) / sum(slots_total) > 0.85 for 15m
                                 OR increase(reservations_rejected_total[1h]) > 0
```

A single rejected reservation is a provider that could not join the network. That is a critical, not
a warning.

**3. A ninth runbook — `relay-capacity-expansion.md`** — distinct from `relay-node-replacement.md`.
Replacement restores capacity; expansion adds it. They are different procedures with different
urgency and different failure modes, and MVP §8.5's eight-file list conflates them by omission.

**4. `architecture.md §27.5` gains a measured-vs-assumed column** on the CGNAT fraction, updated
from `vyomanaut_cluster_relay_dependent_providers` once private beta has 30+ days of data — the same
measure-then-freeze discipline ADR-068 applies to benchmarks and NFR-043 applies to the Postgres
ceiling.

## Consequences

The architecture's nearest ceiling gets a leading indicator with days of lead time instead of a
lagging one with none. Q20-1 — open since the capacity analysis was written, and worth ±280
providers of runway — closes on measurement rather than on a literature assumption. NFR-006's
*"relay overhead < 50 ms RTT"* checklist entry in Phase 18.3, currently marked "— measured" with no
mechanism, gets one.

Cost: relay-mode `/metrics` exposure, four metrics, two alerts, one runbook. This is the cheapest
scalability work in the entire M18 scope and it protects the constraint that binds first.

**Open constraints:**

- The 128 slots/node figure in §27.5 is libp2p Circuit Relay v2's *default* reservation limit, not a
  measured Vyomanaut value. `slots_total` should read the configured limit at runtime rather than
  assume 128 — otherwise the alert divides by a constant that may not be true. Folded into the
  decision above; noted because §27.5's whole table inherits the assumption.
- Whether a relay node should *refuse* new reservations above a soft cap to preserve headroom for
  existing providers is a policy question this ADR does not answer. Q-M18-5.

---
