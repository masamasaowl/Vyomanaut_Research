# ADR-065 — Launch gates are code, not shell thresholds

**Status:** Accepted
**Topic:** #18 Launch Readiness *(new topic)*
**Supersedes:** — *(amends `requirements.md §7.4`, `MVP §8.5`)*
**Source:** This review; `requirements.md §7.4/§7.5`; `architecture.md §27.4`; NFR-043; the
`NetworkProfile` precedent (ADR-031)

## Context

Seven launch-gate thresholds currently live in **four places at once**: `requirements.md §7.4`
(prose), `requirements.md §7.5` (procedure code), `build_part3.md` Session 18.2.1 (a third
restatement, which is where the `p99 ≤ 400 ms` AONT figure appears with no upstream source), and
seven shell scripts that do not yet exist. D-05 and D-13 are both instances of the same failure:
a number restated in four documents drifts in at least one.

The project has already solved this problem once. `NetworkProfile` (ADR-031) exists precisely so
that no timing or shard parameter is written twice, and `TestProfileBothFullySpecified` makes the
Go compiler enforce completeness. Launch thresholds have no equivalent.

There is a second, sharper problem. NFR-043 says the measured Postgres ceiling *"must replace the
5,000–10,000 rows/sec planning estimate in `architecture.md §27.4`"* — a **manual documentation
edit gating GA**, with nothing that fails if it is skipped. `architecture.md §27.1` still carries
the row `ASSUMED — must be benchmarked before V2 GA`. Nothing in CI notices.

## Updates

The B-01 clause is deleted — the demo freeze resolved it permanently. Q-M18-2 resolved: the benchmark result machine schema is fixed as `{cpu_model, cores, base_clock_mhz, aes_ni: bool, sha_ni: bool, ram_gb, disk_class: ssd}`

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — prose thresholds, independent shell scripts** | Zero new code; scripts are runnable on any machine without a Go toolchain | Seven numbers × four locations; D-13 already happened; no artifact survives a run, so "the benchmark passed" is a claim in a build log, not evidence; write-back to §27.4 is manual and unenforced |
| **Thresholds in a YAML/JSON config read by the scripts** | Single source; no Go build needed on the bench machine | Nothing forces the config to be *complete* — a missing threshold silently means "no gate." This is exactly the failure `TestProfileBothFullySpecified` was written to prevent for `NetworkProfile` |
| **`internal/launchgate` Go package: struct literal + result JSON + verdict tool — chosen** | Compiler enforces completeness (all fields in a struct literal, ADR-031's own OR-03 mechanism); one number, one place; each run emits a durable signed-shaped artifact; write-back becomes CI-checkable | Bench machine needs the Go toolchain — acceptable: `aont_encode` and `argon2id` are already Go-path benchmarks, and the minimum-spec machine runs Ubuntu 22.04 |
| **Full benchmark harness (Go `testing.B` + benchstat)** | Idiomatic; statistical rigour for free | `rocksdb_hdd`, `postgres_insert_ceiling` and `e2e_upload` are cross-process, multi-machine measurements that do not fit `testing.B`; would force two mechanisms anyway |

## Decision

**1. `internal/launchgate/gate.go` holds every launch threshold as a single struct literal**, with a
compiler-enforced completeness test mirroring `TestProfileBothFullySpecified`:

```go
// LaunchGate is the complete set of measured launch blockers. Every field is
// required; a zero value is a build failure, not a passing gate.
type LaunchGate struct {
    ID             string          // stable benchmark identifier, e.g. "Q16-1"
    Profile        string          // "prod" | "demo"
    Requirement    string          // "NFR-009"
    InputBytes     int64           // the exact workload size (closes D-13)
    P50Max         time.Duration
    P99Max         time.Duration
    Blocking       bool            // false => advisory, recorded but not gating
    MeasuredSource string          // "" => threshold is provisional, awaiting first run
}
```

**2. Every benchmark emits `scripts/benchmarks/results/{id}.{profile}.json`** in one schema:
`{id, profile, requirement, input_bytes, samples, p50_ms, p99_ms, machine, git_sha, timestamp, verdict}`.
A run that does not produce this file did not happen.

**3. `cmd/vyomanaut-gate` is the only thing that renders a verdict.** Scripts measure; the tool
compares against `LaunchGate` and exits non-zero on any blocking failure or any missing result file.

**4. Measure-then-freeze, applied uniformly.** Where `MeasuredSource` is empty, the first
minimum-spec run *sets* the threshold and the value is committed with the machine spec. This
generalises the NFR-043 pattern the project already accepted, and is how D-13's corrected AONT
threshold gets its number rather than my hand-picking one.

**5. Write-back becomes CI-enforced.** A new grep gate fails if
`scripts/benchmarks/results/NFR-043.prod.json` exists while `architecture.md §27.4` still contains
the string `5,000–10,000`, or while `§27.1`'s ceiling row still reads `ASSUMED`.

**6. `CertifiedAuditPrimitive`** — a frozen constant in the same package, resolving B-01 (§1).

## Consequences

Seven thresholds collapse to one location. D-05 and D-13 become structurally impossible rather than
individually fixed. Every launch certification produces a durable artifact tied to a git SHA and a
machine spec, which is the minimum evidentiary standard for the research paper discussed in your
question 4. The cost is one small Go package and a Go toolchain requirement on the bench machine.

**Open constraints:**

- The `machine` field's schema is not specified here (CPU model, AES-NI presence, disk class, RAM).
  Needs a shape before the first run, or results are not comparable across machines. Q-M18-2.
- `e2e_upload` is inherently multi-machine; its result JSON needs a topology descriptor the other
  six do not. Deliberately deferred to Session 18.2.2's implementation.

---
