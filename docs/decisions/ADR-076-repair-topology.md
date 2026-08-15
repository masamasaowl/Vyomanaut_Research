# ADR-076 — Repair Topology: Build the `r0` Gate, Move Execution Provider-Side

**Status:** Accepted
**Track:** LTS
**Topic:** #4 Replication / Repair Protocol · #3 Confidentiality
**Supersedes:** — *(implements `ADR-004`'s ratified trigger flow, which was never built; restores
`ADR-004`/`ADR-021`'s stated P2P constraint, which the implementation violates)*
**Superseded by:** —
**Research source:** Design Council session 2 of August 2026
(`design-council-2026-08-lts-blockers.md`); live code verification against
`internal/repair/*`, `cmd/microservice/repair_loop.go`, `internal/config/profiles.go`

---

## Context

Two deviations from ADR-004 were found by reading the repair path against the ADR that specifies it.
Neither is recorded anywhere in the codebase, and neither was found by any test.

### F-LTS-07 — the lazy-repair gate does not exist

ADR-004 steps 3–4 gate all repair on a segment's available fragment count falling to `s + r0 = 24`.
No such gate is implemented. Every repair-enqueue call site:

| Call site | Trigger | Gate |
| --- | --- | --- |
| `internal/repair/departure.go:183` | silent departure (72 h) | none — enqueues per chunk |
| `internal/api/provider.go:1409` | announced departure | none — enqueues per chunk |
| `internal/api/admin.go:682` | admin endpoint | operator-supplied count |

`EnqueueRepairForRealChunks` enqueues **one job per ACTIVE non-vetting chunk assignment** on
departure, passing `profile.TotalShards - 1` as `available_shard_count` — a placeholder its own
comment documents: *"neither caller tracks live fragment counts per segment (that bookkeeping
belongs to the audit/threshold-monitoring subsystem, out of scope here)."* **That subsystem does not
exist.** `LazyRepairR0` (prod 8, demo 1) is read in exactly two places — `internal/api/owner.go` for
owner-facing health display and `internal/api/admin.go` for range validation — and never gates a
trigger.

The system therefore performs **eager repair**: the option ADR-004 rejected, at a bandwidth cost
ADR-004 itself computes as `(r − r0)/ln((s+r)/(s+r0)) = 32/0.848 ≈ 38×`. Giroire's `BWavg` of
39 Kbps/peer becomes ~1.5 Mbps/peer against a 100 Kbps background budget — 15× over, before any
correlated-failure variance. This survives today only because the demo profile has five providers
and no churn.

### F-LTS-08 — the microservice reconstructs the AONT package

ADR-004: *"The repair process itself must be P2P — no central entity fetches and re-encodes on
behalf of others."* ADR-021 states the same constraint. ADR-059 cites it as the reason the
microservice cannot tag reconstructed shards.

`cmd/microservice/repair_loop.go:152` calls `repair.ExecuteRepairJob(...)` with
`microservicePeerID`. `internal/repair/executor.go:203`:

```go
aontPackage, err := engine.DecodeSegment(shards)
```

The coordination service downloads `profile.DataShards` (16 in production) shards and decodes them.
**On every repair event, the operator holds a decodable AONT package** — the party ADR-019 says
never sees plaintext. Combined with ADR-059's proposal to hand the microservice per-file
authenticator keys, this becomes a self-serve extraction path: fabricate an audit failure → provider
marked departed → repair triggers → operator decodes. One compromised host, no collusion, no
external attacker.

### The result that frames the decision

Council 2's Systems Theorist established that this is not a bug to be relocated. There are exactly
three candidate repairers, and each is disqualified by a property of the current code family:

| Repairer | Disqualifying property |
| --- | --- |
| A **provider** | must gather `k = 16` shards → obtains the plaintext |
| The **operator** | obtains it, **and** breaks ADR-019's stated trust model, **and** carries all repair egress on one host |
| The **owner** | offline by design (ADR-004) → cannot carry durability |

**No assignment of the repair role under RS + AONT-RS leaves every party blind.** F-69 is therefore
upgraded from a finding to a structural result: either the code family changes (Domain P) or some
party is explicitly trusted with plaintext. This ADR does not resolve that. It resolves which party
holds the exposure *in the meantime*, and reduces how often the exposure occurs.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Ratify the implementation** — eager repair, microservice-executed | Zero work; it already runs | Accepts a 38× bandwidth penalty ADR-004 costed and rejected, routes 793 GB per failure event through one host, and makes the operator the holder of every repaired file's plaintext. Fails ADR-019's central product claim |
| **Restore P2P execution, leave repair eager** | Removes the operator from the plaintext path | Leaves the 38× penalty and makes it *worse* — eager repair means every nightly absence triggers a full reconstruction on a consumer desktop, which is precisely the failure ADR-004's lazy threshold exists to prevent |
| **Build the `r0` gate, leave execution in the microservice** | Removes the bandwidth penalty; smallest change | Leaves the operator decoding every repaired segment. The confidentiality violation is not made acceptable by being rarer |
| **Build the `r0` gate *and* move execution provider-side — chosen** | Both deviations closed; ADR-004 and ADR-021 hold as ratified; repair becomes rare, which is the precondition Domain P's research is evaluated against | Two milestones of work. Requires a per-segment fragment counter, a threshold scan, an elected-repairer protocol, and a fix to `findMissingShardIndex` |
| **Owner-driven repair** | Nobody but the data owner ever holds `k` | Dead on arrival: ADR-004 establishes the owner is offline by design. Durability cannot depend on an owner's login frequency |

## Decision

### 1. Build the `r0 = 8` threshold gate as ADR-004 ratified it

- A **per-segment live fragment count** becomes a maintained quantity — a materialised view or a
  trigger on `chunk_assignments`, decided at implementation. It is the missing
  "audit/threshold-monitoring subsystem" the departure path's comment already names.
- Repair enqueues when a segment's available count reaches `s + r0` (24 in production, 4 in demo),
  **not** on departure. Departure decrements the count; it does not enqueue.
- `available_shard_count` stops being `profile.TotalShards - 1` and carries the real value.
- **`findMissingShardIndex` must be replaced, not reused.** It derives "missing" from a
  caller-supplied surviving-holders list rather than a fresh scan, so a threshold scanner cannot
  call it. The direct-lookup pattern already adopted for `lookupShardIndexForChunk` is the correct
  shape.

ADR-004's scheduler-priority ordering is preserved and its ambiguity resolved in favour of steps 3–4:
permanent departure sets `PriorityPermanentDeparture` on jobs that the **threshold** enqueues; it
does not enqueue independently.

### 2. Repair execution moves to `cmd/provider`

- The repair executor leaves `cmd/microservice`. The microservice retains **election and
  bookkeeping** — choosing the repairing provider and the replacement holder, tracking job state —
  and never receives shard bytes.
- A new protocol elects a repairer and instructs it. The capability-token machinery needed for a
  provider to *write* another provider's shard already exists: tokens bind `chunk_id` and
  `provider_id` (ADR-072, and `segment_id`/`shard_index` under ADR-072 Addendum A), and
  `internal/repair/executor.go` already mints them.
- **The elected repairer obtains the plaintext.** This is stated, not hidden. It is strictly better
  than the operator obtaining it — the exposure moves from one permanent, network-wide party to a
  rotating, per-event one, and it is the exposure ADR-004 and ADR-019 always implied — but it is not
  a fix, and this ADR does not claim it as one.

### 3. Order of work

**The gate first, then the topology.** Eager repair saturates whatever repairer is chosen before the
topology question matters; and the gate alone converts repair from a constant event into a rare one,
which is the regime Domain P's candidate mechanisms must be evaluated against.

### 4. Consequences recorded for downstream ADRs

- **ADR-026 closes as *no V3 repair-bandwidth optimisation*.** With the gate built, every repair
  event reconstructs 32 fragments and every single-block-optimised code family delivers 0%, exactly
  as ADR-004's own table states. Q26-4 is resolved by this ADR; Q62-1 resolves in ADR-026's favour;
  Q62-2 and Q64-1 close unstarted.
- **ADR-059's Q68-3 loses an option.** Microservice-side tagging of reconstructed shards was
  attractive only because the microservice already held both the keys and the bytes. It holds
  neither after this ADR. Q68-3 reduces to a two-way choice — give the repairing provider the
  authenticator keys, or leave reconstructed shards unauditable until the owner returns — and
  neither is acceptable, which makes **R-47 the gating read** for the Proof of Storage milestone.
- **`build_part4.md`'s milestone map changes.** Confidentiality moves ahead of Proof of Storage;
  Repair & Erasure Optimisation is removed as a milestone.

## Consequences

**Positive.** ADR-004 and ADR-021 hold as ratified rather than as aspiration. The 38× bandwidth
penalty is removed. The operator leaves the plaintext path. Repair becomes rare, which is both a
durability-economics win and the precondition that makes Domain P's research tractable — a partial
mitigation is worth much more against a rare event than a constant one.

**Negative.** Two milestones. A new protocol surface (elected repairer) with its own authentication
and failure modes. The elected provider still obtains plaintext, so the structural problem is
relocated and reduced, not solved.

**Open constraints:**

- **The repairing provider obtains a decodable package.** Named, not fixed. → Domain P (R-27, R-28,
  R-29), and specifically R-28 (repair without any party assembling `k`), which is now load-bearing.
- **Repairer election is a new attack surface.** A provider that can influence its own election
  gains plaintext on demand. Election must not be predictable or petitionable. → Q76-1.
- **Fragment-count maintenance cost is unmeasured.** A trigger on `chunk_assignments` at
  5,000 providers × 5,000 files is a write-amplification question against NFR-043's Postgres
  ceiling. → LaunchGate measurement, Q76-2.
- **The demo is unaffected and stays unaffected.** ADR-062 freezes it at `demo-v1.0.0`; this ADR is
  `Track: LTS` and must not be backported.

## References

- ADR-004 — the trigger flow this ADR implements and the P2P constraint it restores; its Q26-4
  ambiguity is resolved here
- ADR-019, ADR-021 — the trust and transport constraints F-LTS-08 violates
- ADR-026 — closes as a consequence of this decision
- ADR-059 — Q68-3's option set is narrowed by §4
- ADR-072 + Addendum A — the capability-token binding the elected-repairer protocol reuses
- `design-council-2026-08-lts-blockers.md` Council 2 — the three-repairers result
- `reading-list.md` §5 Domain P — the research this ADR makes tractable
