# ADR-031 — Demo / Production Mode: Single Codebase, Profile-Driven Configuration

**Status:** Accepted  
**Topic:** #0 Cross-cutting — Build Strategy and Demo Topology  
**Supersedes:** ADR-029 `--dev-mode` flag (for demonstration purposes; ADR-029 simulation mode remains valid for CI)  
**Superseded by:** —  
**Research source:** mvp.md (May 2026); ADR-029 (simulation mode precedent)

---

## Context

Vyomanaut V2 needs to be demonstrable at investor meetings and hackathons before the full
production network (56 providers, 3 relay nodes, secrets manager, live Razorpay integration)
is operational. The risk with a "demo build" is that it diverges from production, so bugs
found during demo testing do not surface real production bugs, and code proved in demo is not
the code that runs in production.

ADR-029 set the precedent: simulation mode must run the same binary as production. This ADR
extends that principle to the demo scenario.

---

## Decision

### 1. The Mode Flag

A single environment variable controls the active mode:

```
VYOMANAUT_MODE=demo | prod
```

A CLI flag `--mode=demo | --mode=prod` overrides the environment variable. The default is
`prod`. The mode is read once at process startup, stored as a package-level constant, and
printed as the first log line:

```
[STARTUP] Vyomanaut daemon v0.1.0 — mode=DEMO — do not use for real data
[STARTUP] Vyomanaut daemon v0.1.0 — mode=PRODUCTION
```

The mode cannot be changed without restarting the process. No API endpoint or runtime signal
alters it.

**Mandatory startup guard rails:**

- If `VYOMANAUT_MODE=prod` AND `VYOMANAUT_CLUSTER_MASTER_SEED` is present → **fatal, refuse to start.** Secrets manager is required in production.
- If `VYOMANAUT_MODE=demo` AND the process connects to the live Razorpay API endpoint → **fatal, refuse to start.** Real money must not move during a demo.
- If `VYOMANAUT_MODE` is absent → default to `prod`, log a warning.

### 2. The NetworkProfile Struct

All mode-variable parameters live in a single struct. No `if mode == "demo"` conditionals appear in business logic. The mode is used only for startup guard rails and the startup log line.

```go
// internal/config/network_profile.go

type NetworkProfile struct {
    Mode string // "demo" or "prod" — printed at startup, never used for branching

    // Erasure coding (ADR-003)
    DataShards   int
    ParityShards int
    TotalShards  int
    ShardSize    int // FIXED: 262144 in both modes; must not change
    LazyRepairR0 int

    // Readiness gate (ADR-029)
    MinActiveProviders int
    MinDistinctASNs    int
    MinMetroRegions    int
    MinRelayNodes      int
    MinCooledAccounts  int

    // ASN cap (ADR-014)
    ASNCapFraction float64

    // Time windows
    HeartbeatInterval          time.Duration
    HeartbeatJitter            time.Duration
    PollingInterval            time.Duration
    DHTRepublishInterval       time.Duration
    DHTExpiryDuration          time.Duration
    DepartureThreshold         time.Duration
    PromisedDowntimeMaximum    time.Duration
    AuditPeriodDuration        time.Duration
    EscrowHoldWindow           time.Duration
    VettingHoldWindow          time.Duration
    PendingReceiptGCAge        time.Duration
    RepairPromotionTimeout     time.Duration

    // Scoring windows (ADR-008)
    ScoreWindowShort  time.Duration
    ScoreWindowMedium time.Duration
    ScoreWindowLong   time.Duration
    DualWindowDrop    float64 // always 0.20; never mode-variable

    // Vetting (ADR-005)
    VettingMinPasses   int
    VettingMinDuration time.Duration
    VettingCapFraction float64 // always 0.10; never mode-variable

    // Cryptographic cost (ADR-020)
    Argon2Time    uint32
    Argon2Memory  uint32 // KiB
    Argon2Threads uint8

    // Infrastructure
    RequireSecretsManager      bool
    RequireQuorum              bool
    PaymentMode                string        // "razorpay_live" | "razorpay_test" | "mock"
    SkipMnemonicConfirm        bool
    RazorpayCoolingPeriod      time.Duration

    // Release computation
    ReleaseComputationInterval time.Duration // 0 = calendar-date driven (prod)

    // Vetting GC retry backoff (§4.5 interface-contracts)
    GCRetryBackoff []time.Duration
}
```

**Parameters that must never appear in NetworkProfile (identical in both modes):**

- `ShardSize` (262,144) — compile-time constant; changing it breaks vLog sizing, audit framing, and RocksDB assumptions simultaneously
- Challenge nonce length (33 bytes) — wire-format constant
- `ASNCapFraction` decimal (0.20) — the percentage is fixed; only `TotalShards` varies
- `VettingCapFraction` (0.10) — fixed policy
- `DualWindowDrop` (0.20) — fixed policy
- All cipher identities (ChaCha20, AES-CTR, AEAD_CHACHA20_POLY1305, Ed25519, SHA-256, HMAC-SHA256, HKDF)
- Poly1305 constant-time tag comparison — always enforced
- Row security policies on `audit_receipts` and `escrow_events`
- The single-writer vLog goroutine requirement

### 3. The Two Profile Instances

```go
// internal/config/profiles.go

var ProductionProfile = NetworkProfile{
    Mode:         "prod",
    DataShards: 16, ParityShards: 40, TotalShards: 56,
    ShardSize: 262144, LazyRepairR0: 8,
    MinActiveProviders: 56, MinDistinctASNs: 5, MinMetroRegions: 3,
    MinRelayNodes: 3, MinCooledAccounts: 56,
    ASNCapFraction: 0.20,
    HeartbeatInterval: 4 * time.Hour, HeartbeatJitter: 5 * time.Minute,
    PollingInterval: 24 * time.Hour, DHTRepublishInterval: 12 * time.Hour,
    DHTExpiryDuration: 24 * time.Hour, DepartureThreshold: 72 * time.Hour,
    PromisedDowntimeMaximum: 72 * time.Hour, AuditPeriodDuration: 30 * 24 * time.Hour,
    EscrowHoldWindow: 30 * 24 * time.Hour, VettingHoldWindow: 60 * 24 * time.Hour,
    PendingReceiptGCAge: 48 * time.Hour, RepairPromotionTimeout: 6 * time.Hour,
    ScoreWindowShort: 24 * time.Hour, ScoreWindowMedium: 7 * 24 * time.Hour,
    ScoreWindowLong: 30 * 24 * time.Hour, DualWindowDrop: 0.20,
    VettingMinPasses: 80, VettingMinDuration: 120 * 24 * time.Hour,
    VettingCapFraction: 0.10,
    Argon2Time: 3, Argon2Memory: 65536, Argon2Threads: 4,
    RequireSecretsManager: true, RequireQuorum: true,
    PaymentMode: "razorpay_live", SkipMnemonicConfirm: false,
    RazorpayCoolingPeriod: 24 * time.Hour,
    ReleaseComputationInterval: 0,
    GCRetryBackoff: []time.Duration{5 * time.Minute, 15 * time.Minute, 60 * time.Minute},
}

var DemoProfile = NetworkProfile{
    Mode:         "demo",
    DataShards: 3, ParityShards: 2, TotalShards: 5,
    ShardSize: 262144, LazyRepairR0: 1,
    MinActiveProviders: 5, MinDistinctASNs: 5, MinMetroRegions: 1,
    MinRelayNodes: 0, MinCooledAccounts: 5,
    ASNCapFraction: 0.20,
    HeartbeatInterval: 30 * time.Second, HeartbeatJitter: 5 * time.Second,
    PollingInterval: 2 * time.Minute, DHTRepublishInterval: 2 * time.Minute,
    DHTExpiryDuration: 4 * time.Minute, DepartureThreshold: 10 * time.Minute,
    PromisedDowntimeMaximum: 10 * time.Minute, AuditPeriodDuration: 2 * time.Minute,
    EscrowHoldWindow: 1 * time.Minute, VettingHoldWindow: 2 * time.Minute,
    PendingReceiptGCAge: 5 * time.Minute, RepairPromotionTimeout: 3 * time.Minute,
    ScoreWindowShort: 2 * time.Minute, ScoreWindowMedium: 6 * time.Minute,
    ScoreWindowLong: 20 * time.Minute, DualWindowDrop: 0.20,
    VettingMinPasses: 5, VettingMinDuration: 5 * time.Minute,
    VettingCapFraction: 0.10,
    Argon2Time: 1, Argon2Memory: 4096, Argon2Threads: 1,
    RequireSecretsManager: false, RequireQuorum: false,
    PaymentMode: "mock", SkipMnemonicConfirm: true,
    RazorpayCoolingPeriod: 0,
    ReleaseComputationInterval: 2 * time.Minute,
    GCRetryBackoff: []time.Duration{10 * time.Second, 30 * time.Second, 2 * time.Minute},
}
```

### 4. Profile Injection

The profile is passed via dependency injection to every subsystem constructor. No subsystem reads `VYOMANAUT_MODE` directly.

```go
func selectProfile() config.NetworkProfile {
    mode := os.Getenv("VYOMANAUT_MODE")
    if flag.Lookup("mode") != nil {
        mode = *modeFlag
    }
    switch mode {
    case "demo":
        log.Printf("[STARTUP] Vyomanaut — mode=DEMO — do not use for real data")
        return config.DemoProfile
    case "prod", "":
        if mode == "" {
            log.Printf("[STARTUP] WARNING: VYOMANAUT_MODE not set; defaulting to prod")
        }
        log.Printf("[STARTUP] Vyomanaut — mode=PRODUCTION")
        return config.ProductionProfile
    default:
        log.Fatalf("[STARTUP] FATAL: unknown VYOMANAUT_MODE=%q", mode)
    }
    panic("unreachable")
}
```

### 5. Corrected Demo ASN Constraint

The demo `MinDistinctASNs` is **5** (not 2, which was an error in a prior analysis). With
`TotalShards=5` and `ASNCapFraction=0.20`, each ASN may hold `floor(5 × 0.20) = 1` shard.
Satisfying a 1-shard-per-ASN constraint across 5 shards requires exactly 5 distinct ASNs.
Setting `MinDistinctASNs` to anything below 5 would make the 20% cap impossible to satisfy
and violate ADR-014. In `--sim-count=5 --sim-asn-count=5` mode (ADR-029), this is automatic.

### 6. Database Schema Parameterisation

Two CHECK constraints vary by profile. They are emitted by a migration generator:

```go
// migrations/generator.go
func GenerateInitialSchema(profile config.NetworkProfile) string {
    return fmt.Sprintf(`
CONSTRAINT chunk_assignments_shard_index_range
    CHECK (shard_index BETWEEN 0 AND %d OR shard_index IS NULL),
CONSTRAINT repair_jobs_shard_count_range
    CHECK (available_shard_count BETWEEN %d AND %d),
`,
        profile.TotalShards-1,
        profile.DataShards,
        profile.TotalShards,
    )
}
```

The `mv_provider_scores` materialised view is dropped and recreated at microservice startup
using the profile's scoring window values — it is not a migration-time operation.

### 7. Mandatory Compiler-Enforced Tests

```go
func TestProfileShardSizeIsConstant(t *testing.T) {
    require.Equal(t, config.ProductionProfile.ShardSize, config.DemoProfile.ShardSize,
        "ShardSize must be 262144 in both profiles — changing it breaks wire format")
}

func TestProfilesBothFullySpecified(t *testing.T) {
    // Go struct-literal syntax enforces that every field is set in both profiles.
    // This test confirms neither profile uses a zero-value default accidentally.
    require.NotZero(t, config.DemoProfile.PollingInterval)
    require.NotZero(t, config.ProductionProfile.PollingInterval)
    // ... repeat for every field that must not be zero
}
```

### 8. Recommended Build Order

Build against `DemoProfile` first. The entire end-to-end lifecycle (register → vet → upload → audit → repair → pay → depart) is observable in ~35 minutes with demo parameters. Switching to production is changing the active profile and wiring three pieces of infrastructure (secrets manager, Razorpay live, relay nodes). No Go function changes. No schema changes beyond the two parameterised CHECK constraint values.

**Phase 0** — Scaffold NetworkProfile, both profile instances, startup guard rails, compiler tests, two separate databases (demo\_db, prod\_db).

**Phase 1** — Core pipeline with DemoProfile: crypto, erasure, storage, p2p, provider daemon, microservice, data owner client. Milestone: full upload/download cycle on one laptop in < 5 minutes.

**Phase 2** — Provider lifecycle: vetting, repair, scoring, payment (mock), vetting GC. Milestone: full lifecycle demo in 35 minutes.

**Phase 3** — Production infrastructure: SecretsManagerClient, RazorpayProvider, 3 relay nodes, (3,2,2) gossip cluster. Milestone: `VYOMANAUT_MODE=prod` passes all readiness conditions.

**Phase 4** — Hardening: benchmarks Q16-1, Q18-1, Q27-1, HDD audit latency, Razorpay integration tests, disaster recovery runbook. Milestone: private beta.

---

## Demo Network Specifications (reference)

| Parameter | Production | Demo |
|---|---|---|
| DataShards (s) | 16 | 3 |
| ParityShards (r) | 40 | 2 |
| TotalShards (n) | 56 | 5 |
| ShardSize (lf) | 262,144 B | 262,144 B (unchanged) |
| LazyRepairR0 | 8 | 1 |
| Lazy repair trigger (s+r0) | 24 | 4 |
| Emergency floor (s) | 16 | 3 |
| Max segment size | 14 MB | 1.25 MB |
| Fault tolerance | 40-of-56 | 2-of-5 |
| Min active providers | 56 | 5 |
| Min distinct ASNs | 5 | 5 |
| Min metro regions | 3 | 1 |
| Min relay nodes | 3 | 0 |
| Heartbeat interval | 4 h | 30 s |
| Polling/audit interval | 24 h | 2 min |
| Departure threshold | 72 h | 10 min |
| Vetting passes required | 80 | 5 |
| Vetting minimum duration | 120 days | 5 min |
| Escrow hold window (post-vetting) | 30 days | 1 min |
| Release computation cycle | Monthly (23rd) | Every 2 min |
| Argon2id (t, m, p) | 3, 64 MB, 4 | 1, 4 MB, 1 |
| Secrets manager | Required | Env var substitute |
| Payment gateway | Razorpay live | Mock in-memory |
| Microservice replicas | 3 (quorum) | 1 (no quorum) |
| Razorpay cooling | 24 h | 0 s |
| BIP-39 mnemonic confirmation | Required | Skipped |

---

## Consequences

**Positive:**
- Full production code path exercised from day one; demo bugs are production bugs
- End-to-end lifecycle observable in 35 minutes; vetting/scoring/repair bugs found in hours not months
- Switching to production is pure infrastructure wiring — no logic changes
- Go struct-literal syntax enforces that every field is specified in both profiles; zero-value defaults are compile-time visible

**Negative / trade-offs:**
- Demo Argon2id parameters (t=1, m=4 MB) are significantly weaker against offline brute-force; acceptable only because demo data is not real
- `MinDistinctASNs=5` in demo means all 5 demo providers must declare distinct synthetic ASNs; this requires either `--sim-asn-count=5` or manual `--demo-asn=SIM-AS{N}` at each provider startup
- The `mv_provider_scores` view is regenerated at every microservice restart; a restart during a demo session briefly shows stale scores

**Open constraints:**
- ADR-029's `--dev-mode` flag for quorum bypass in simulation is superseded by `VYOMANAUT_MODE=demo` for demonstration purposes; CI simulation continues to use `--dev-mode` as specified in ADR-029
- The mock `PaymentProvider` must be reviewed at each Razorpay API update to confirm it continues to exercise the correct interface surface

## Affected Documents

| Document | Change |
|---|---|
| `docs/system-design/architecture.md` | Add §9 principle "Profile-driven configuration", §4 Technology Stack row, §8 and §24 notes |
| `docs/system-design/requirements.md` | Add §5.10 Mode Requirements; notes on FR-053 and §7.1 |
| `docs/system-design/data-model.md` | Parameterise two CHECK constraints; add mv_provider_scores note; two migration checklist items |
| `docs/system-design/interface-contracts.md` | Update §3.4 readiness thresholds; §5.1 Argon2id injection; §5.2 erasure injection; §6 chunk_assignments note; §10 versioning clause |
| `docs/api/openapi.yaml` | Readiness condition threshold descriptions; optional `demo_asn` on provider register |

## References

- [ADR-029](ADR-029-bootstrap-minimum-viable-network.md): simulation mode precedent; `--sim-asn-count=5` satisfies the corrected `MinDistinctASNs=5` for demo
- [ADR-003](ADR-003-erasure-coding.md): RS parameters overridden by NetworkProfile in demo
- [ADR-005](ADR-005-peer-selection.md): VettingMinPasses and VettingMinDuration overridden
- [ADR-006](ADR-006-polling-interval.md): polling and departure thresholds overridden
- [ADR-008](ADR-008-reliability-scoring.md): scoring window values overridden
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap enforced in both modes; `MinDistinctASNs=5` corrects the demo value required to make the cap satisfiable
- [ADR-020](ADR-020-key-management.md): Argon2id parameters overridden
- [ADR-024](ADR-024-economic-mechanism.md): escrow hold windows and release cycle overridden
- [ADR-027](ADR-027-cluster-audit-secret.md): secrets manager replaced by env var in demo
- [mvp.md](./mvp.md): full demo timeline, viability fact-checks, and build order analysis