# ADR-029 — Bootstrap Minimum Viable Network and Provider Daemon Simulation Mode

**Status:** Accepted
**Topic:** #1 Coordination Architecture, #5 Peer Selection
**Supersedes:** —
**Superseded by:** —
**Research source:** ADR-003 (RS(16,56) reconstruction threshold), ADR-005 (ASN cap and
provider pool), ADR-014 (20% ASN cap), ADR-025 ((3,2,2) microservice quorum),
Paper 10 — Giroire (BWavg and LossRate at scale), Paper 36 — Dalle et al. (correlated
failure variance), Paper 38 — Nath et al. (ASN cap as reliability requirement),
answered question Q30-1 (relay infrastructure sizing)

---

## Context

No ADR specifies the minimum conditions under which the system may begin accepting data owner
uploads. Starting uploads before the network satisfies the RS(16,56) reconstruction threshold
(at least 56 active providers), the ASN diversity requirement (at least 5 distinct ASNs), and
the microservice quorum (ADR-025) would:

1. Make every upload un-encodable (fewer than 56 providers → cannot place all 56 shards)
2. Invalidate the 20% ASN cap (ADR-014) — Paper 38 proves the cap is a co-requisite for the
   LossRate < 10⁻¹⁵ guarantee; without 5+ ASNs, the cap provides no diversity
3. Produce repair events with nowhere to place replacement shards

Additionally, no ADR describes how to test the system at realistic scale without access to
thousands of physical machines. The testing strategy is a first-class design decision: if the
simulation mode is an afterthought, engineers will implement features that cannot be tested
locally, and integration bugs will only surface in production.

---

## Decision — Part A: Minimum Viable Network Conditions

The microservice assignment service MUST return HTTP 503 ("Network not ready") for all
data owner upload requests until **all** of the following conditions are simultaneously true:

| Condition | Threshold | Rationale |
|---|---|---|
| Active vetted providers | ≥ 56 | RS(16,56) requires exactly 56 distinct shard holders per file. This is a hard constraint from ADR-003. |
| Distinct ASNs in active pool | ≥ 5 | With 5 ASNs, the 20% cap allows each ASN ≤ 11 shards — non-trivial diversity. With < 5 ASNs, one ASN necessarily holds > 20%. Paper 38 proves the cap is a reliability co-requisite. |
| Distinct Indian metro regions | ≥ 3 | Geographic baseline: Delhi NCR, Mumbai, and one southern metro (Bangalore / Hyderabad / Chennai). Prevents all providers clustering in one city. |
| Microservice cluster state | Full (3,2,2) quorum | ADR-025. Degraded quorum (2 of 3 replicas) is operational but not the bootstrap baseline. |
| Razorpay Linked Accounts ready | ≥ 56 providers with 24h cooling period complete | Paper 35 (Razorpay API): Linked Accounts have a 24-hour cooling period before the first transfer. No provider can receive payment until this completes. |
| Relay infrastructure | ≥ 3 relay nodes deployed | Q30-1 (answered): 3 relay nodes (one per Indian AZ) provides 384 simultaneous relay slots — 4.3× headroom at 300 initial providers. |
| Cluster audit secret | server_secret_v1 loaded on all replicas | ADR-027: all replicas must share the cluster secret before any challenges are issued. |

The upload gate is enforced as a readiness check in the assignment service startup path and
re-evaluated every 60 seconds. The readiness check is also exposed as a health endpoint
(`GET /api/v1/admin/readiness`) for monitoring dashboards.

**Why shard count is not the bootstrap metric:**
ADR-005 previously specified "5 × n shards" (280 shards) as the ASN cap activation threshold.
This is incorrect. Shards are a per-file count, not a network-wide count. At 56 providers and
1 file, the network has exactly 56 shards — but the cap is already enforceable on that file.
The correct threshold is expressed in terms of provider count and ASN count, not shard count.
The "5 × n shards" clause in ADR-005 is superseded by the conditions in this ADR.

---

## Decision — Part B: Provider Daemon Simulation Mode

The provider daemon MUST support multi-instance simulation mode from day 1. This is not a
test harness bolted on at QA time — it is a first-class daemon feature, tested in CI, and
used at every stage of development.

### CLI invocation

```bash
# Run 56 simulated provider instances on a single machine
vyomanaut-provider \
  --sim-count=56 \
  --sim-base-port=4001 \
  --sim-data-dir=/tmp/vyomanaut-sim \
  --microservice-url=http://localhost:8080
```

Each simulated instance gets:
- A distinct Ed25519 key pair (generated at first startup, persisted under
  `/tmp/vyomanaut-sim/{instance_id}/keys/`)
- A distinct RocksDB instance at `/tmp/vyomanaut-sim/{instance_id}/db/`
- A distinct vLog file at `/tmp/vyomanaut-sim/{instance_id}/vlog/chunks.vlog`
- A distinct libp2p listen address at `127.0.0.1:{base_port + instance_index}`
- A distinct `provider_id` registered with the microservice

Simulated instances share nothing except the microservice endpoint. The daemon process
manages all N instances in a single process using goroutines — one goroutine per instance
for the audit challenge handler, one shared heartbeat scheduler with per-instance timers.

### Simulation ASN metadata

In simulation mode, each instance is registered with a synthetic ASN drawn from a round-robin
assignment:

```
sim_asn = "SIM-AS" + ((instance_index % sim_asn_count) + 1)
sim_region = ["Delhi", "Mumbai", "Bangalore"][instance_index % 3]
```

`--sim-asn-count=5` (default) produces 5 synthetic ASNs, satisfying the ≥ 5 ASN bootstrap
condition. The assignment service treats synthetic ASNs identically to real ASNs for cap
enforcement purposes.

### Scale targets and required infrastructure

| Node count | Purpose | Infrastructure |
|---|---|---|
| 56 | Minimum viable network verification. RS(16,56), 5 ASN cap, repair scheduler basic test. | 1 laptop. Simulation mode, no network simulation. |
| 100 | ASN cap enforcement and repair scheduler integration test. 5 ASNs, 100 providers. | 2 laptops. Simulation mode. |
| 1,000 | DHT lookup latency, repair bandwidth, and heartbeat storm tests under realistic network conditions. | Linux network namespaces + `tc netem` with 20–80ms inter-provider RTT injected to simulate Indian metro-to-metro ISP latency. |
| 1,000,000 | Analytical validation only — simulation is not feasible at this scale. | Verify Giroire formulas (Paper 10) with N=1M substituted. |

### Analytical validation at N = 1,000,000

From Giroire Formula 1 (Paper 10), BWavg scales as B/N (total blocks / provider count).
Total blocks B = D / lb = D / 4MB. For D = 1 exabyte stored at N = 1,000,000 providers:

```
B = 1e18 / 4e6 = 2.5e11 blocks
BWavg ≈ (2.5e11 × α) / (1e6 × ln(56/24)) × 47 × 256KB
       ≈ 39 Kbps/peer   (same as N=1,000 — BWavg is invariant with N for fixed D/N ratio)
```

BWavg is proportional to D/N, not N. Provided the per-provider data volume (D/N) is held
constant, the network bandwidth budget is scale-invariant. LossRate at N=1M is also safe:
LossRate ∝ B × (α/γ)^(r0+2). With r0=8, the exponential term dominates and LossRate remains
< 10⁻¹⁵ per year regardless of N. No parameter changes are required at V3+ scale.

### Simulation mode and the readiness gate

In simulation mode, the readiness gate (Part A) is **not bypassed**. The simulation must
satisfy the same conditions as production:
- `--sim-count` must be ≥ 56
- `--sim-asn-count` must be ≥ 5
- The microservice must be running at full quorum (can be a single instance in dev mode
  with quorum checks disabled via `--dev-mode` flag)

This ensures that tests written against simulation mode are valid proxies for production
behaviour.

---

## Consequences

**Positive:**
- Every engineer can run a 56-node network on their laptop from day 1 — no shared test
  infrastructure needed for basic integration tests
- The readiness gate prevents silent misconfiguration where uploads fail non-obviously due
  to an under-populated provider pool
- The 5-ASN minimum is now expressed as a first-class constraint in the assignment service
  code, not in documentation
- Scale-up path is explicit: laptop → 2 laptops → namespace simulation → analytical — no
  surprise at each stage

**Negative / trade-offs:**
- Multi-instance simulation mode adds daemon complexity: resource isolation between instances
  must be correct or test results are invalid (a vLog path collision between two instances
  produces undefined behaviour)
- The readiness gate is a binary check; there is no "degraded mode" upload path for operators
  who have 50 providers and want to start testing. This is intentional — partial compliance
  with RS(16,56) is not a valid operating mode

**Open constraints:**
- The simulation mode's goroutine-per-instance model must be load-tested to confirm that
  56 concurrent audit challenge handlers do not contend in ways that a single provider would
  not — goroutine scheduling on a laptop does not faithfully represent independent OS
  processes on separate machines
- Linux network namespace tooling for the 1,000-node test target needs a runbook; this is
  an engineering specification task, not an ADR task

---

## Summary: ASN Cap Threshold Correction (Supersedes ADR-005 clause)

ADR-005 states: *"Correlated failure prevention (Honest Geppetto): Activate only after
5 × n shards exist in the network."*

This clause is superseded by Part A of this ADR. The correct activation condition is:

> The 20% ASN cap is enforced on every chunk assignment from the moment the network satisfies
> the bootstrap readiness conditions in ADR-029: ≥ 56 vetted providers across ≥ 5 distinct ASNs.
> The cap is never deactivated. The "5 × n shards" clause in ADR-005 is retired.

---

## References

- [ADR-003](ADR-003-erasure-coding.md): RS(16,56) — 56 providers required per file; the fundamental reason for the 56-provider minimum
- [ADR-005](ADR-005-peer-selection.md): ASN cap activation — the incoherent "5 × n shards" clause superseded here
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap — confirmed as both a security and reliability co-requisite
- [ADR-025](ADR-025-microservice-consistency-mechanism.md): (3,2,2) quorum — a bootstrap prerequisite
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): BWavg and LossRate formulas; N=1M analytical validation
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): BWavg is mean only; ASN cap bounds the variance that the Giroire formula does not capture
- [Paper 38 — Nath et al.](../research/paper-38-nath-correlated-failures.md): 5 ASNs minimum is the threshold below which the 20% cap provides no meaningful diversity — proven by the diminishing return finding for large-m erasure schemes
- [answered Q30-1](../research/answered-questions.md): 3 relay nodes minimum; sizing rationale
