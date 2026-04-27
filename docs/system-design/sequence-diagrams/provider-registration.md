# Provider Registration and Onboarding

**ADR references:** [ADR-001](../../decisions/ADR-001-coordination-architecture.md) · [ADR-005](../../decisions/ADR-005-peer-selection.md) · [ADR-007](../../decisions/ADR-007-provider-exit-states.md) · [ADR-021](../../decisions/ADR-021-p2p-transfer-protocol.md) · [ADR-024](../../decisions/ADR-024-economic-mechanism.md) · [ADR-028](../../decisions/ADR-028-provider-heartbeat.md) · [ADR-029](../../decisions/ADR-029-bootstrap-minimum-viable-network.md)  
**Architecture reference:** [architecture.md §12 Provider Lifecycle](../architecture.md#12-provider-lifecycle)

---

## Overview

A provider goes through five distinct phases before becoming fully eligible for chunk assignments: **installation**, **registration**, **payment rail setup**, **vetting**, and **activation**. Each phase has explicit status transitions and failure states. Physical row deletion is prohibited at every stage — departures are soft-deleted only ([ADR-007](../../decisions/ADR-007-provider-exit-states.md), [ADR-013](../../decisions/ADR-013-consistency-model.md)).

---

## Phase 1 — Installation and Key Generation

```
Provider Machine                        File System (~/.vyomanaut/)
─────────────────────────────────────────────────────────────────────
     │                                          │
     │── installer runs ─────────────────────► │
     │                                          │
     │   daemon first-launch detection:         │
     │   keys/ directory absent?                │
     │                                          │
     │   YES → generate Ed25519 key pair        │
     │          using OS CSPRNG                 │
     │          (go crypto/ed25519)             │
     │                                          │
     │── encrypted keypair ──────────────────► keys/provider.key
     │   (encrypted under daemon passphrase,    │   (Ed25519 private key,
     │    HKDF-derived keystore enc key)         │    AES-256-GCM encrypted)
     │                                          │
     │── public key fingerprint ─────────────► keys/provider.pub
     │   displayed to provider for records      │   (32-byte Ed25519 public key,
     │                                          │    hex-encoded, plaintext)
     │                                          │
     │   Peer ID derived:                       │
     │   peer_id = multihash(public_key)        │
     │   (libp2p convention, ADR-021)           │
     │                                          │
     │── daemon starts libp2p host ──────────► binds listen ports
     │   QUIC v1 primary (RFC 9000)             │   QUIC: udp/4001
     │   TCP + Noise XX fallback                │   TCP:  tcp/4001
     │                                          │
     │   AutoNAT probe:                         │
     │   classify: public / cone / symmetric    │
     │                                          │
     │   IF symmetric NAT:                      │
     │     reserve Circuit Relay v2 slot        │
     │     at nearest Vyomanaut relay node      │
     │     (ADR-021, three-tier NAT traversal)  │
     │                                          │
```

---

## Phase 2 — OTP Verification and Registration

```
Provider App/UI       OTP Service          Microservice            Postgres
──────────────────────────────────────────────────────────────────────────────
     │                    │                    │                       │
     │── POST /api/v1/auth/otp/send ─────────►│                       │
     │   { phone_number,                       │                       │
     │     purpose: PROVIDER_REGISTER }        │                       │
     │                                         │── send SMS OTP ──────►│ (external)
     │◄── 200 { expires_at } ─────────────────│                       │
     │                                         │                       │
     │   (provider enters 6-digit OTP)         │                       │
     │                                         │                       │
     │── POST /api/v1/auth/otp/verify ────────►│                       │
     │   { phone_number, otp_code }            │                       │
     │                                         │── verify OTP ─────────│
     │                                         │                       │
     │   FAILURE PATH:                         │                       │
     │   OTP invalid / expired                 │                       │
     │◄── 401 INVALID_OTP ────────────────────│                       │
     │   provider must re-request OTP          │                       │
     │   (rate-limited: 3 attempts / 10 min)   │                       │
     │                                         │                       │
     │   SUCCESS PATH:                         │                       │
     │◄── 200 { token (short-lived JWT),  ────│                       │
     │          is_new_entity: true,           │                       │
     │          role: null }                   │                       │
     │                                         │                       │
     │── POST /api/v1/provider/register ──────►│                       │
     │   Authorization: Bearer <token>         │                       │
     │   {                                     │                       │
     │     ed25519_public_key: "<32-byte hex>",│                       │
     │     declared_storage_gb: 500,           │── INSERT providers ──►│
     │     city: "Mumbai",                     │   status =            │
     │     region: "Mumbai",                   │   PENDING_ONBOARDING  │
     │     asn: "AS24560",                     │                       │
     │     initial_multiaddrs: [...],          │                       │
     │     provider_sig: "<Ed25519 sig>"       │                       │
     │   }                                     │                       │
     │                                         │                       │
     │   FAILURE PATH:                         │                       │
     │   phone already registered              │                       │
     │◄── 409 PHONE_ALREADY_REGISTERED ───────│                       │
     │                                         │                       │
     │   FAILURE PATH:                         │                       │
     │   provider_sig fails verification       │                       │
     │◄── 403 INVALID_BODY_SIGNATURE ─────────│                       │
     │                                         │                       │
     │   SUCCESS PATH:                         │                       │
     │◄── 201 {                           ────│                       │
     │     provider_id: "<UUID>",              │                       │
     │     status: PENDING_ONBOARDING,         │                       │
     │     token: "<7-day JWT>",               │                       │
     │     razorpay_cooling_until:             │                       │
     │       "<NOW + 24h>"                     │                       │
     │   }                                     │                       │
     │                                         │                       │
```

---

## Phase 3 — Razorpay Linked Account Setup (Async)

```
Microservice                    Razorpay Route API              Postgres
────────────────────────────────────────────────────────────────────────────
     │                               │                               │
     │   (triggered immediately      │                               │
     │    after registration INSERT) │                               │
     │                               │                               │
     │── POST /v1/accounts ─────────►│                               │
     │   {                           │                               │
     │     profile.name: provider_id,│                               │
     │     type: "route",            │                               │
     │     legal_business_name: ..., │                               │
     │     business_type: "individual"                               │
     │   }                           │                               │
     │                               │                               │
     │   FAILURE PATH:               │                               │
     │   Razorpay API error          │                               │
     │◄── 4xx / 5xx ────────────────│                               │
     │   retry up to 3×              │                               │
     │   with exponential back-off   │                               │
     │   if all retries fail:        │                               │
     │   flag for manual review      │                               │
     │                               │                               │
     │   SUCCESS PATH:               │                               │
     │◄── 200 { id: "acc_XXXXX" } ──│                               │
     │                               │                               │
     │── UPDATE providers ───────────────────────────────────────────►│
     │   SET razorpay_linked_account_id = "acc_XXXXX"                │
     │       razorpay_cooling_until   = NOW() + INTERVAL '24 hours'  │
     │   WHERE provider_id = ?        │                               │
     │                               │                               │
     │   ┌─ 24-hour cooling window ──────────────────────────────┐   │
     │   │  Razorpay requires 24h before any transfer to a new   │   │
     │   │  Linked Account (Paper 35, ADR-024).                  │   │
     │   │  Chunk assignment is BLOCKED until:                   │   │
     │   │    razorpay_cooling_until < NOW()                     │   │
     │   │    AND razorpay_linked_account_id IS NOT NULL         │   │
     │   └───────────────────────────────────────────────────────┘   │
     │                               │                               │
```

---

## Phase 4 — First Heartbeat and Status Transition

```
Provider Daemon               Microservice                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                            │                               │
     │   daemon boots, 4-hour     │                               │
     │   heartbeat timer starts   │                               │
     │   (ADR-028)                │                               │
     │                            │                               │
     │── POST /api/v1/provider/heartbeat ────────────────────────►│
     │   Authorization: Bearer <token>                            │
     │   {                        │                               │
     │     provider_id: "<UUID>", │                               │
     │     current_multiaddrs: [  │                               │
     │       "/ip4/.../udp/4001   │                               │
     │        /quic-v1/p2p/...",  │                               │
     │       "/ip4/.../tcp/4001   │                               │
     │        /p2p/..."           │                               │
     │     ],                     │                               │
     │     timestamp: "<ISO8601>",│                               │
     │     daemon_version: "0.1.0"│                               │
     │     provider_sig: "<sig>"  │                               │
     │   }                        │                               │
     │                            │── verify Ed25519 sig ─────────│
     │                            │   against stored public key   │
     │                            │                               │
     │   FAILURE PATH:            │                               │
     │   timestamp > 5 min skew   │                               │
     │◄── 400 CLOCK_SKEW ─────────│                               │
     │   daemon re-syncs clock    │                               │
     │   and retries              │                               │
     │                            │                               │
     │   FAILURE PATH:            │                               │
     │   provider status=DEPARTED │                               │
     │◄── 403 PROVIDER_DEPARTED ──│                               │
     │   daemon halts; re-join    │                               │
     │   requires new registration│                               │
     │                            │                               │
     │   SUCCESS PATH:            │                               │
     │                            │── UPDATE providers ───────────►│
     │                            │   SET last_heartbeat_ts = NOW()│
     │                            │       last_known_multiaddrs    │
     │                            │         = [multiaddrs]        │
     │                            │       multiaddr_stale = false  │
     │                            │       status = 'VETTING'       │
     │                            │         (if was               │
     │                            │          PENDING_ONBOARDING)   │
     │                            │                               │
     │◄── 200 {               ────│                               │
     │     received_at: "...",    │                               │
     │     microservice_sig: "..."│                               │
     │   }                        │                               │
     │                            │                               │
     │   daemon persists          │                               │
     │   microservice_sig as      │                               │
     │   proof of acknowledgement │                               │
     │                            │                               │
     │   4-hour timer resets;     │                               │
     │   heartbeat repeats every  │                               │
     │   4h indefinitely          │                               │
     │   (ADR-028)                │                               │
     │                            │                               │
```

---

## Phase 5 — Vetting Period

```
Audit Scheduler     Provider Daemon        Microservice            Postgres
────────────────────────────────────────────────────────────────────────────
     │                    │                    │                       │
     │   (4–6 month       │                    │                       │
     │    vetting period) │                    │                       │
     │                    │                    │                       │
     │   Assignment service assigns            │                       │
     │   non-critical chunks under             │                       │
     │   extra erasure redundancy              │                       │
     │   (ADR-005)         │                   │                       │
     │                    │                    │                       │
     │   Each 24h window:  │                   │                       │
     │── audit challenge ─►│                   │                       │
     │   (challenge_nonce, │                   │                       │
     │    chunk_id)        │                   │                       │
     │                    │── vLog read ───────│                       │
     │                    │   verify           │                       │
     │                    │   content_hash     │                       │
     │                    │   compute          │                       │
     │                    │   response_hash    │                       │
     │                    │                    │                       │
     │                    │── POST /audit/receipt ────────────────────►│
     │                    │   (signed receipt) │── INSERT              │
     │                    │                    │   audit_receipts      │
     │                    │◄── 200 { service_sig } ───────────────────│
     │                    │                    │                       │
     │                    │                    │── UPDATE providers ───►│
     │                    │                    │   SET consecutive_     │
     │                    │                    │   audit_passes += 1   │
     │                    │                    │   (reset to 0 on      │
     │                    │                    │    any FAIL/TIMEOUT)  │
     │                    │                    │                       │
     │   Vetting economics:│                   │                       │
     │   Hold window: 60 days (not 30)         │                       │
     │   Release cap: 50% of monthly earnings  │                       │
     │   (ADR-024)         │                   │                       │
     │                    │                    │                       │
```

---

## Phase 6 — Vetting Completion and Activation

```
Microservice                                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                                               │
     │   After each PASS receipt is countersigned:   │
     │   check consecutive_audit_passes              │
     │                                               │
     │   IF consecutive_audit_passes >= 80           │
     │   (Jeffrey's prior β(0.5,0.5) threshold,      │
     │    Paper 05, ADR-005):                        │
     │                                               │
     │── UPDATE providers ───────────────────────────►│
     │   SET status = 'ACTIVE'                       │
     │                                               │
     │   Escrow mechanics revert:                    │
     │   hold window: 30 days (was 60)               │
     │   release cap: removed (was 50%)              │
     │   (ADR-024)                                   │
     │                                               │
     │   Provider now receives full chunk            │
     │   assignment rate via Power of Two Choices    │
     │   weighted by reliability score (ADR-005)     │
     │                                               │
```

---

## Status Transition Map

```
                    ┌──────────────────────────┐
                    │   daemon install +        │
                    │   key pair generated      │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │  POST /provider/register  │
                    │  OTP verified             │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                   ┌─────────────────────────────┐
                   │   PENDING_ONBOARDING          │
                   │   Razorpay account creating   │
                   │   24h cooling in progress     │
                   └──────────┬──────────┬────────┘
                              │          │
                    cooling   │          │  no heartbeat
                    elapsed + │          │  within 24h of
                    heartbeat │          │  registration
                    received  │          │
                              ▼          ▼
                   ┌──────────────┐   STALLED
                   │  VETTING     │   (daemon must
                   │  (4–6 months)│   be restarted)
                   └──────┬───┬──┘
                          │   │
         80 consecutive   │   │  >50% audits fail
         passes (Jeffrey  │   │  during vetting
         prior threshold) │   │
                          ▼   ▼
                   ┌──────────────┐   FAILED_VETTING
                   │   ACTIVE     │   (escrow forfeited;
                   │  (full rate) │   new phone required
                   └──────┬───┬──┘   to re-register)
                          │   │
         announced        │   │  72h no contact
         departure        │   │  (silent departure)
                          ▼   ▼
                   ┌──────────────────────────┐
                   │        DEPARTED           │
                   │  (soft delete only —      │
                   │   row never removed,      │
                   │   ADR-007, ADR-013)       │
                   └──────────────────────────┘
                          │
                          │  reconnects after
                          │  DEPARTED?
                          ▼
                   HTTP 403 PROVIDER_DEPARTED
                   re-join requires new
                   registration + new phone
```

---

## Heartbeat Miss Escalation

```
Provider Daemon               Microservice (departure detector)
──────────────────────────────────────────────────────────────
     │  (missed heartbeat)         │
     │                             │
     │   0–8h (1 missed):          │
     │   no action; within         │
     │   nightly absence window    │
     │                             │
     │   8–12h (2 missed):         │
     │                             │── UPDATE providers
     │                             │   SET multiaddr_stale = true
     │                             │   Challenge dispatch now uses
     │                             │   DHT fallback (ADR-028)
     │                             │
     │   12–24h (3 missed):        │
     │                             │── DHT FIND_NODE lookup
     │                             │   attempt address resolution
     │                             │   log failure if DHT also stale
     │                             │
     │   ≥ 72h:                    │
     │   (ADR-006, ADR-007)        │── SET status = DEPARTED
     │                             │── DELETE chunk_assignments
     │                             │   (stops challenge issuance)
     │                             │── Seize escrow → repair reserve
     │                             │── Queue repair jobs for all chunks
     │                             │   (see sequence diagram 04)
     │                             │
```

---

## Key Invariants

| Invariant | Source |
|---|---|
| Provider rows are never physically deleted | [ADR-007](../../decisions/ADR-007-provider-exit-states.md), [ADR-013](../../decisions/ADR-013-consistency-model.md) |
| Chunk assignment blocked until Razorpay cooling elapsed | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), Paper 35 |
| Ed25519 body signature verified on every heartbeat | [ADR-028](../../decisions/ADR-028-provider-heartbeat.md) |
| `consecutive_audit_passes` resets to 0 on any FAIL/TIMEOUT | [ADR-005](../../decisions/ADR-005-peer-selection.md) |
| Vetting hold window is 60 days (not 30), release capped at 50% | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) |
| `DEPARTED` provider reconnecting → HTTP 403; new phone to re-join | [ADR-007](../../decisions/ADR-007-provider-exit-states.md) |