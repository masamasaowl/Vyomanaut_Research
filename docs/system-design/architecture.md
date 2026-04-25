# Vyomanaut V2 — System Architecture

**Status:** Design-complete. Research phase closed. Build phase beginning.  
**Last updated:** April 2026  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
**Related:** All decisions in `docs/decisions/`, all research in `docs/research/`

---

## 1. What This Document Is

This document describes how Vyomanaut V2 is built. It covers every major component, how they connect, and why each design choice was made. Every significant decision points back to an Architecture Decision Record (ADR) in `docs/decisions/`.

Read this document before writing a single line of code. If you find a gap between this document and a specific ADR, the ADR wins — it was written later with more context. If you find a gap between this document and the research notes, raise it before building.

---

## 2. What Vyomanaut Does

Vyomanaut is a paid distributed storage network. Data owners pay to store their files. Storage providers — people running the Vyomanaut daemon on their home desktop or NAS — earn money for keeping those files available and passing regular storage audits.

Three things make Vyomanaut different from services like Dropbox or Google Drive:

**The service never sees the data.** Files are encrypted on the data owner's device before anything leaves. The providers store encrypted pieces. The microservice coordinates the system. Nobody in the middle can read anything stored. ([ADR-019](../decisions/ADR-019-client-side-encryption.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md))

**Data is split across 56 independent providers.** A file is broken into 56 encrypted fragments, and any 16 of them are enough to reconstruct the original. This means a provider can go offline, disappear, or fail — the data owner is unaffected as long as enough fragments survive. ([ADR-003](../decisions/ADR-003-erasure-coding.md))

**Payment is tied to proof, not trust.** Providers are paid only when they pass an audit — a daily challenge where they prove they still hold the data they were assigned. They cannot fake a pass without the actual data. ([ADR-002](../decisions/ADR-002-proof-of-storage.md), [ADR-012](../decisions/ADR-012-payment-basis.md))

---

## 3. What V2 Does Not Do

Understanding the scope prevents building the wrong things:

- **No mobile providers in V2.** Home desktops and NAS devices only. Phones have OS-enforced background execution limits that make storage unreliable. Mobile support is explicitly deferred to V3. ([ADR-010](../decisions/ADR-010-desktop-only-v2.md))
- **No blockchain.** All coordination, auditing, and payment runs through a hardened microservice backed by Postgres and Razorpay. The three things blockchains provide in competing systems (immutable log, automatic payment trigger, public dispute resolution) are replicated without on-chain writes. ([ADR-015](../decisions/ADR-015-audit-trail.md))
- **No deduplication.** Every file upload uses a fresh random encryption key. Files are not deduplicated across different data owners.
- **No file versioning or in-place updates.** Files are immutable once stored. If a data owner wants to update a file, they delete the old one and upload the new one.
- **No retrieval market.** Payment is for proving storage availability, not for serving downloads. A provider earns whether or not anyone retrieves their data.

---

## 4. Core Design Principles

These principles governed every design decision. When a new engineering choice comes up during the build, test it against these before deciding.

**Lazy everything.** The system defers work whenever possible. Repair is deferred until redundancy actually drops to a threshold — not triggered immediately on every failure. This alone reduces repair bandwidth by roughly 38 times compared to reacting to every departure. ([ADR-004](../decisions/ADR-004-repair-protocol.md), [ADR-006](../decisions/ADR-006-polling-interval.md))

**Prove, don't trust.** No provider is trusted to report their own state accurately. Storage is verified by challenge-response audits. Bandwidth throughput is measured during audits, not taken from declarations. The microservice issues all challenges; providers cannot influence when they are tested. ([ADR-002](../decisions/ADR-002-proof-of-storage.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md))

**Coordinate only where necessary.** Most operations — recording an audit pass, registering a new file, appending to the audit log — can happen on any replica without coordination. Six operations genuinely require a single authoritative source: decrementing an escrow balance, enforcing a reliability score floor, assigning chunks uniquely, validating capability tokens, and seizing escrow. Only those six go through the coordinated path. Everything else scales horizontally. ([ADR-013](../decisions/ADR-013-consistency-model.md))

**Bound correlated failures structurally.** When providers fail in correlated ways — same ISP, same city, same power grid — a naive redundancy scheme can lose more data than the math predicts. The system prevents this by ensuring no single correlated group holds more than 20% of any file's fragments at assignment time. This cap is both a security defence and a reliability requirement. ([ADR-014](../decisions/ADR-014-adversarial-defences.md), [ADR-003](../decisions/ADR-003-erasure-coding.md))

**Pay for presence, not transfer.** Providers earn per audit passed, not per gigabyte downloaded by data owners. This decouples the payment system from the P2P transfer layer — a microservice outage cannot accumulate credit liability. ([ADR-012](../decisions/ADR-012-payment-basis.md))

---

## 5. System Map

The system has four planes that operate somewhat independently:

```
┌─────────────────────────────────────────────────────────────┐
│  DATA OWNER (desktop app / web UI)                          │
│  Encrypts locally → uploads fragments → holds pointer file  │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS (upload, retrieval, account)
┌────────────────────▼────────────────────────────────────────┐
│  COORDINATION MICROSERVICE  (3 replicas, Postgres)          │
│  Registration · Audit scheduling · Payment · Repair queue   │
│  Gossip membership · Kademlia DHT republication             │
└──────┬──────────────────────────────────────────┬───────────┘
       │ Audit challenges (QUIC / libp2p)         │ DHT lookups
┌──────▼──────────┐                    ┌──────────▼────────────┐
│  PROVIDER DAEMON │  ←── libp2p ────► │  PROVIDER DAEMON       │
│  (desktop/NAS)  │     (QUIC v1)      │  (desktop/NAS)         │
│  WiscKey storage│                    │  WiscKey storage        │
│  Heartbeat      │                    │  Heartbeat              │
└─────────────────┘                    └────────────────────────┘
         ×56 total providers per file, distributed across India
```

**Coordination plane:** The microservice cluster handles all orchestration. It does not touch data — it only knows which chunk is on which provider, who passed their last audit, and who should be paid. Three replicas with a (3, 2, 2) quorum (3 replicas, 2 needed to read, 2 needed to write) ensure one failure does not interrupt operation. ([ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md))

**Data plane:** Actual chunks move directly between the data owner's device and the providers over P2P connections using libp2p and QUIC. The microservice is never in the data path. ([ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md))

**Audit plane:** The microservice sends daily challenge messages to each provider. The provider computes a response that can only be computed correctly if they hold the actual chunk. The result is recorded in an append-only audit log. ([ADR-002](../decisions/ADR-002-proof-of-storage.md), [ADR-017](../decisions/ADR-017-audit-receipt-schema.md))

**Payment plane:** A Razorpay account holds data owner deposits. As providers pass audits, their earned balance accumulates in the microservice's internal ledger. Once a month, the portion eligible for release is transferred to the provider's UPI-linked bank account. ([ADR-011](../decisions/ADR-011-escrow-payments.md), [ADR-024](../decisions/ADR-024-economic-mechanism.md))

---

## 6. The Encoding Pipeline

Before any data leaves the data owner's device, it passes through a four-stage pipeline. Nothing that touches the network is readable without the data owner's credentials.

### Stage 1 — Client-side encryption (AONT transform)

The data owner's file is processed by the All-or-Nothing Transform (AONT). This works as follows: a fresh random 256-bit key K is generated for the segment. Every 16-byte word of the segment is XORed with a keystream produced from K. A commitment hash h = SHA-256(all encrypted words) is computed, and the final block is set to K XOR h. This embeds the key inside the data.

The security property: an attacker who holds fewer than k=16 fragments cannot compute h (because they are missing pieces), therefore cannot recover K, therefore cannot decrypt any word. You need all 16 reconstruction-threshold fragments to get the key. ([ADR-022](../decisions/ADR-022-encryption-erasure-order.md))

The cipher is ChaCha20-256 on hardware without AES acceleration (most Indian desktops), and AES-256-CTR on hardware with AES-NI. The daemon detects which is available at startup. The hash is SHA-256 throughout. ([ADR-019](../decisions/ADR-019-client-side-encryption.md))

### Stage 2 — Reed-Solomon erasure coding

The AONT package is fed into Reed-Solomon coding with parameters s=16, r=40, producing 56 fragments of 256 KB each. The parameters mean: any 16 of the 56 fragments are enough to reconstruct the original. The other 40 are parity.

These parameters are not arbitrary. The Giroire formulas (from research Paper 10) show that r=40 is the mathematically optimal redundancy level for minimising repair bandwidth when s=16 and the lazy repair threshold is r0=8. The resulting steady-state repair bandwidth is approximately 39 Kbps per provider — well inside the 100 Kbps background budget. ([ADR-003](../decisions/ADR-003-erasure-coding.md))

### Stage 3 — Fragment distribution

The 56 fragments are uploaded to 56 distinct providers selected by the assignment service. The assignment service enforces: no single ASN (autonomous system number — roughly an ISP) may hold more than 20% of any file's fragments (about 11 of 56). This prevents an ISP outage or a coordinated group of bad actors from destroying a file. ([ADR-005](../decisions/ADR-005-peer-selection.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md))

### Stage 4 — Pointer file

After upload completes, the data owner holds a pointer file: a structured record containing the list of 56 providers, the 56 chunk content addresses, and the erasure parameters. The pointer file is encrypted with AEAD_CHACHA20_POLY1305 before being stored anywhere outside the owner's device. The microservice stores the encrypted ciphertext but cannot decrypt it.

The pointer file is the only retrieval credential. Losing it with no backup means permanent data loss — this is disclosed clearly at onboarding. ([ADR-020](../decisions/ADR-020-key-management.md))

---

## 7. Key Hierarchy

The data owner's credentials form a three-level hierarchy, all derived from a single master secret:

```
passphrase + owner_id
      │
      ▼  Argon2id (t=3, m=64 MB, p=4)
 master_secret (32 bytes, never stored on disk)
      │
      ├─► HKDF("vyomanaut-file-v1" || file_id)  ──► file_key (per file)
      │
      └─► HKDF("vyomanaut-pointer-v1" || file_id) ─► pointer file encryption key
                                                          │
                                             AEAD_CHACHA20_POLY1305
                                                          │
                                                    encrypted pointer file
                                                    (stored by microservice)
```

The AONT key K is not part of this hierarchy. It is embedded inside the fragments themselves and recovered automatically when 16 fragments are assembled. The data owner never manages K directly.

A 24-word BIP-39 mnemonic can be generated from the master secret at account creation for offline backup. Recovery from mnemonic restores the master secret and from it all file keys. ([ADR-020](../decisions/ADR-020-key-management.md))

---

## 8. Provider Lifecycle

A provider moves through five states before reaching full operation, and through one of four exit states when leaving.

### 8a. Joining

**Step 1 — Registration.** The provider downloads the daemon, which generates an Ed25519 key pair on first run. The provider registers via the app, providing their phone number (OTP-verified) and their public key. The microservice creates a provider record with status `PENDING_VERIFICATION`. ([ADR-001](../decisions/ADR-001-coordination-architecture.md))

**Step 2 — Razorpay Linked Account setup.** Simultaneously, the microservice creates a Razorpay Route Linked Account for the provider. This account has a mandatory 24-hour cooling period before the first payment transfer can occur. No chunk assignments happen during this window.

**Step 3 — First heartbeat.** Once the daemon is running, it sends a signed heartbeat to the microservice control plane every 4 hours. The heartbeat contains the provider's current network address (multiaddr), which changes when their home router gets a new IP. The microservice uses this address for all future audit challenges — not the DHT. ([ADR-028](../decisions/ADR-028-provider-heartbeat.md))

**Step 4 — Vetting begins.** The provider receives non-critical chunk assignments under extra erasure redundancy. Audits run daily. For the first 4–6 months, earnings accumulate on a 60-day hold window with a 50% release cap. This gives the network time to verify reliability before trusting a provider with critical data.

**Step 5 — Full operation.** After 80+ consecutive successful audits (the statistical threshold at which a 99% confidence estimate of the provider's reliability is achievable), the provider is promoted to `ACTIVE`. The hold window shortens to 30 days and the release cap is lifted. ([ADR-005](../decisions/ADR-005-peer-selection.md), [ADR-024](../decisions/ADR-024-economic-mechanism.md))

### 8b. During operation

While active, the provider:
- Sends a signed heartbeat every 4 hours with their current network address ([ADR-028](../decisions/ADR-028-provider-heartbeat.md))
- Receives audit challenge messages from the microservice roughly once per day per stored chunk
- Stores incoming chunks in the WiscKey storage engine ([ADR-023](../decisions/ADR-023-provider-storage-engine.md))
- Earns per audit passed; earnings accumulate in the microservice's internal escrow ledger ([ADR-012](../decisions/ADR-012-payment-basis.md))

### 8c. Exiting

Four exit states with different financial consequences: ([ADR-007](../decisions/ADR-007-provider-exit-states.md))

| State | What happened | Repair trigger | Escrow outcome |
|---|---|---|---|
| Temporary absence | Gone < 72 hours, no notice | None — wait | Score decremented per polling cycle |
| Promised downtime | Announced planned absence in advance | None — wait for promise period | Fine if promise broken |
| Silent departure | Gone > 72 hours, no announcement | Repair triggered immediately | All held escrow seized for repair fund |
| Announced departure | Provider explicitly notified the system | Repair triggered immediately | Held escrow released proportionally |

The 72-hour threshold comes directly from Bolosky's measurement of desktop computer absence patterns: 99.7% of weekend absences (the longest normal absence) resolve within 70 hours. Setting the threshold at 72 hours eliminates false-positive repair triggers from routine weekend downtime while catching real departures. ([ADR-006](../decisions/ADR-006-polling-interval.md))

---

## 9. P2P Transfer

Data moves between providers and data owners over direct peer-to-peer connections, not through the microservice.

**Framework:** libp2p handles peer identity, connection management, multiplexing, and NAT traversal. ([ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md))

**Primary transport:** QUIC v1 (RFC 9000). Each chunk transfer is one independent QUIC stream. A lost packet on one stream does not stall other streams. When a provider's IP address changes (common on Indian residential ISPs with 24-hour DHCP rotations), the connection survives via QUIC's Connection ID mechanism — the connection is named by an identifier, not by the IP address.

**Fallback transport:** TCP with Noise protocol security and yamux multiplexing. This activates automatically when a provider is behind a UDP-blocking middlebox. Empirical measurement shows TCP and QUIC achieve statistically identical hole-punch success rates (~70%), so the fallback carries no reliability penalty.

**NAT traversal — three tiers:**

1. AutoNAT: on connect, peers probe whether the provider is publicly reachable, behind a home-router NAT (punchable), or behind symmetric NAT (relay required).
2. DCUtR: for home-router NAT, a relay node coordinates simultaneous dial attempts from both sides. 97.6% of successful connections succeed on the first attempt. Retry limit is set to 1 (not the default 3), saving 200–800 ms per connection.
3. Circuit Relay v2: for symmetric NAT (roughly 30% of providers), all traffic routes through a Vyomanaut-operated relay node. Three relay nodes in Indian cloud availability zones are required at launch.

**Session resumption:** 0-RTT (zero round-trip reconnect) is disabled for any connection carrying an audit challenge or signed receipt, because 0-RTT data can be replayed. It may be enabled for pure chunk data transfers where replay causes no security consequence.

**Peer identity:** Each provider has an Ed25519 key pair generated at installation. The libp2p Peer ID is derived from this key. The microservice verifies the public key at registration. Every transport connection authenticates the remote Peer ID cryptographically — a provider cannot impersonate another.

**DHT configuration:**
- k-bucket size: k=16 (from the S/Kademlia disjoint-path design)
- Parallel lookups: alpha=3
- All desktop providers run as DHT servers (full participants), not DHT clients
- DHT lookup keys are pseudonymised: `HMAC-SHA256(chunk_hash, file_owner_key)` — the DHT never sees the real chunk identity or file identity

---

## 10. Audit System

The audit system is how the microservice verifies that providers actually hold the chunks they claim to hold.

### 10a. The challenge

Each day, the microservice sends each provider an audit challenge for each chunk they are assigned. The challenge contains:
- The chunk ID
- A nonce: `HMAC(server_secret, chunk_id + server_timestamp)`

The server_secret is shared across all three microservice replicas via HKDF derivation from a cluster master seed stored in a secrets manager. The nonce is versioned so that any replica can validate any challenge, even during a replica failover. ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md))

### 10b. The response

The provider looks up the chunk in their local WiscKey storage, reads the 256 KB chunk data, verifies the content hash to detect silent disk corruption, then computes:

```
response_hash = SHA256(chunk_data || challenge_nonce)
```

Without the actual chunk data, this hash cannot be computed. The response deadline is `(chunk_size / measured_p95_upload_throughput) × 1.5`. This makes it impossible for a provider to retrieve the data just-in-time from another source — the round trip would take longer than the deadline. ([ADR-014](../decisions/ADR-014-adversarial-defences.md))

### 10c. The receipt

The microservice records a 12-field signed receipt in the audit_receipts table. The table is INSERT-only — no row is ever updated or deleted. Both the provider and the microservice sign the receipt with Ed25519. The provider's signature covers all fields; the microservice's countersignature covers the provider's signature and the countersign timestamp.

Key fields: ([ADR-017](../decisions/ADR-017-audit-receipt-schema.md))

| Field | Purpose |
|---|---|
| challenge_nonce | Verifies the response was for this specific challenge (prevents replay) |
| server_challenge_ts | Set by the server — prevents backdating |
| response_hash | The provider's proof they hold the data |
| response_latency_ms | Just-in-time detection: anomalously fast responses flag possible data retrieval at audit time |
| audit_result | PASS / FAIL / TIMEOUT |
| provider_sig | Provider's Ed25519 signature over all fields |
| service_sig | Microservice's countersignature — locks in the result |

### 10d. Audit reliability score

Each provider has three rolling reliability windows: 24 hours, 7 days, and 30 days. The 24-hour window is weighted highest. The final score is a weighted combination of the three.

If the 7-day score drops more than 0.20 below the 30-day score, the system flags this as a deterioration signal and applies a reduced release multiplier on the next monthly payout — even if the provider has not yet hit the 72-hour departure threshold. This catches providers who are degrading before they disappear. ([ADR-008](../decisions/ADR-008-reliability-scoring.md))

---

## 11. Repair System

When a provider goes offline long enough to be declared departed (72 hours), or when a chunk's available fragment count drops to the lazy repair threshold, the repair scheduler acts.

**The repair trigger threshold r0=8.** Repair fires when the number of available fragments for a chunk drops to s + r0 = 24. This is 8 above the reconstruction floor of 16. The system does not wait for fragment count to hit the floor — that would leave too little margin. ([ADR-004](../decisions/ADR-004-repair-protocol.md))

**What repair does.** The repair scheduler contacts 16 surviving fragment holders, downloads their fragments, reconstructs the original AONT package, re-encodes into 56 fragments, and uploads the missing fragments to newly selected providers. The new providers are selected with the same ASN diversity constraints as original assignments.

**Priority ordering.** When the repair queue has multiple jobs, confirmed silent departures (providers past the 72-hour threshold) are processed before pre-warning jobs (chunks approaching but not yet at r0). This prevents permanently degraded chunks from accumulating behind lower-priority work. ([ADR-004](../decisions/ADR-004-repair-protocol.md))

**Bandwidth math.** For 1,000 providers, a single provider failure produces approximately 793 GB of total network transfer during repair. At 100 Kbps per provider, this completes in about 8 hours — inside the 12-hour safety window. The per-peer steady-state repair bandwidth is approximately 39 Kbps, well inside the 100 Kbps background budget. ([ADR-004](../decisions/ADR-004-repair-protocol.md))

---

## 12. Provider-Side Storage Engine

Each provider daemon stores chunks using a WiscKey-style key-value separation design: the chunk index lives in RocksDB; the actual chunk data lives in a separate append-only value log (vLog). ([ADR-023](../decisions/ADR-023-provider-storage-engine.md))

**Why this design.** Standard LSM-trees (like raw RocksDB or LevelDB) store keys and values together and compact them during background work. At 256 KB chunk sizes, this produces 10–14× write amplification — every chunk written eventually causes 10–14× that amount of disk I/O. With WiscKey separation, write amplification drops to approximately 1.0 at 256 KB because values never move during compaction.

**How it works.**

The RocksDB index maps `chunk_id (32 bytes) → (vlog_offset, chunk_size) (12 bytes)`. The index entries are small enough to fit entirely in RocksDB's block cache after a warm-up period. Bloom filters are enabled: an audit challenge for a chunk not assigned to this provider hits only the Bloom filter in memory, with no disk I/O.

The vLog is a single append-only file. Each entry is 262,212 bytes:
- 32 bytes: chunk_id (copy for GC validation)
- 4 bytes: chunk_size
- 262,144 bytes: the chunk data (256 KB)
- 32 bytes: content_hash = SHA256(chunk_data)

The content_hash is verified on every read before computing the audit response. Silent disk corruption — where the disk returns wrong data without reporting an error — surfaces as an audit FAIL rather than a wrong hash being returned to the microservice. ([ADR-023](../decisions/ADR-023-provider-storage-engine.md))

**Concurrent writes.** All vLog appends are serialised through a single writer goroutine. Multiple upload goroutines submit write requests via a buffered channel. This is required because POSIX append atomicity does not hold for writes larger than ~4 KB, and chunk entries are 262 KB.

**Crash recovery.** On restart, the daemon reads the last vLog head pointer from RocksDB and scans the vLog from that point forward, re-inserting any index entries that were written to the vLog but not yet flushed to RocksDB.

**Garbage collection.** GC runs only when chunks are explicitly deleted (provider exit, file owner deletion, repair reassignment). For a write-once cold storage workload, GC is rare.

**Audit lookup path (end to end):**
1. Receive `(chunk_id, challenge_nonce)` from microservice
2. Check Bloom filter — absent → return FAIL immediately (no disk I/O)
3. Read `(vlog_offset, chunk_size)` from RocksDB (typically from block cache — no disk I/O)
4. Read 262,212 bytes from vLog at `vlog_offset` — one random read: ~1 ms on SSD, ~12–15 ms on HDD
5. Verify `SHA256(chunk_data) == content_hash`
6. Compute `response_hash = SHA256(chunk_data || challenge_nonce)`
7. Return signed audit receipt

Both SSD and HDD read times are well within the audit deadline.

---

## 13. Payment System

### 13a. What Razorpay does and does not do

Razorpay's Escrow+ product requires NBFC registration and a trustee. Vyomanaut does not qualify. Instead, the system uses:

- **Smart Collect 2.0**: each data owner gets a virtual UPI ID. When they deposit funds to pay for storage, Razorpay automates reconciliation.
- **Razorpay Route**: each provider is a Linked Account. The microservice creates transfers to provider accounts with `on_hold: true`, then releases them monthly by updating `on_hold_until`.
- **RazorpayX Payouts**: the microservice issues monthly payouts to providers' registered bank accounts via IMPS or UPI.

**UPI Collect is deprecated as of February 2026.** All deposit flows must use UPI Intent (payer selects their UPI app from a list) or QR code. The backend deposit reconciliation via Smart Collect 2.0 is unaffected.

**Idempotency:** Every payout API call includes the `X-Payout-Idempotency` header. The value is `SHA256(provider_id + audit_period)`, taken directly from the `idempotency_key` column in the `escrow_events` table. This prevents double-payment if the network call is retried. ([ADR-016](../decisions/ADR-016-payment-db-schema.md))

### 13b. Internal escrow ledger

The microservice maintains an append-only `escrow_events` table. Balance is never stored as a column — it is always computed as:

```
balance = SUM(DEPOSIT amounts) - SUM(RELEASE + SEIZURE amounts)
          for a given provider_id
```

This is a CRDT PN-counter: deposits and releases are both appended, never updated. ([ADR-016](../decisions/ADR-016-payment-db-schema.md))

### 13c. Release schedule

On the 23rd of each month, the microservice computes each provider's releasable balance. The release multiplier is determined by their 30-day reliability score:

| 30-day score | Release multiplier |
|---|---|
| ≥ 0.95 | 1.00 (full release) |
| 0.80 – 0.95 | 0.75 |
| 0.65 – 0.80 | 0.50 |
| < 0.65 | 0.00 (hold in full) |

Withheld amounts from partial releases are not seized — they roll into the next month's window. They are only seized if the provider subsequently departs silently. ([ADR-024](../decisions/ADR-024-economic-mechanism.md))

Route releases on the next business day after `on_hold_until`. The release window for providers is "within the first 3 business days of each month" to account for bank holidays.

### 13d. Escrow seizure

When a provider is declared silently departed (72 hours without contact):
1. The microservice freezes the provider's account.
2. All earnings in the 30-day rolling window are transferred to the repair reserve fund.
3. A seizure event is appended to `escrow_events` (SEIZURE type).
4. Repair is triggered for all chunks the departed provider held.

The seized escrow funds the cost of onboarding replacement providers.

### 13e. Vetting period economics

During the 4–6 month vetting period, the hold window is 60 days (double the post-vetting window) and the release cap is 50%. This acts as a temporal entry cost — not a financial stake — that deters rapid Sybil registration and immediate withdrawal.

---

## 14. Coordination Microservice

### 14a. What it does

The microservice is the system's control plane. It handles:
- Provider registration and identity verification
- Chunk assignment (which provider holds which fragment)
- Audit challenge scheduling and dispatch
- Audit receipt recording
- Reliability score maintenance
- Repair job queuing
- Payment release computation and execution
- DHT record republication (every 12 hours per provider)

It does not touch file data at any point.

### 14b. Cluster configuration

Three replicas with a (3, 2, 2) quorum: 3 replicas, 2 must respond for a read, 2 must acknowledge a write. One replica can fail without interrupting service. ([ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md))

Membership uses gossip: each replica contacts one randomly chosen peer per second to reconcile membership histories. Two seed nodes (also full replicas) are configured at deploy time to prevent the cluster from splitting into isolated groups on restart.

For the latency-sensitive paths (audit challenge dispatch, chunk assignment), the service uses client-driven routing: the client caches cluster membership and routes directly to the responsible replica, bypassing the load balancer. This removes 30+ ms at the 99.9th percentile compared to load-balancer indirection.

Background tasks (materialised view refresh, repair job queuing, Merkle log compaction) are throttled against the foreground DB read latency. If the 99th-percentile foreground latency approaches 50 ms, background work is slowed down. This keeps audit challenge SLAs intact under maintenance load.

### 14c. The six coordinated operations

Out of approximately 20 core database operations, six cannot be distributed because they involve invariants that two independent replicas cannot maintain without talking to each other. These six go through the single authoritative payment microservice: ([ADR-013](../decisions/ADR-013-consistency-model.md))

1. Decrementing escrow balance (payment release — floor must not go below zero)
2. Seizing escrow (floor enforcement)
3. Enforcing reliability score floor (score cannot go below zero)
4. Assigning a chunk to a unique provider (no two providers assigned the same slot)
5. Validating capability tokens (expiry is time-based and non-distributable)
6. Physical provider row deletion (never done — soft delete only)

The other 14 operations — including all audit receipt inserts, all reputation score increments, all DHT republications — are coordination-free and scale horizontally without limit.

### 14d. Audit trail

All audit receipts are written to an INSERT-only Postgres table. No row is ever updated or deleted, enforced by Postgres row security policy. This table is the tamper-evident audit log that replaces the blockchain's immutable record function. ([ADR-015](../decisions/ADR-015-audit-trail.md))

A V3 upgrade path (Transparent Merkle Log) will allow any party to independently verify the audit history without trusting the microservice operator.

---

## 15. Adversarial Defences

Five classes of adversarial provider behaviour are defended against. ([ADR-014](../decisions/ADR-014-adversarial-defences.md))

**Correlated shard placement (Honest Geppetto attack).** A group of colluding providers accumulates months of genuine trust, then deletes all their data simultaneously. Defence: no single ASN may hold more than 20% of any file's fragments (about 11 of 56). Even if the entire correlated group disappears at once, 45 fragments remain — 29 above the reconstruction threshold of 16.

**Outsourcing.** A provider does not store the data but retrieves it just-in-time from another provider who does. Defence: the audit deadline is `(chunk_size / p95_measured_upload_throughput) × 1.5`. The p95 throughput is measured empirically during the vetting period and updated as an EWMA from audit responses — a provider cannot extend their deadline by self-reporting a low speed. Retrieving 256 KB from another provider over the internet takes longer than the deadline.

**Just-in-time retrieval.** A provider caches data only when a challenge is about to arrive. Defence: challenge timing is randomised using `HMAC(server_secret, chunk_id + server_timestamp)`. Providers cannot predict when the next challenge will arrive.

**False audit responses.** A provider sends a plausible-looking hash without holding the data. Defence: the correct response is `SHA256(chunk_data || challenge_nonce)`. Without the actual chunk data, this cannot be computed. The microservice independently has the expected response hash and verifies it.

**Service denial.** A provider stores the data and passes all audits but refuses to serve retrieval requests from data owners. Defence: RS(16,56) requires more than 40 of 56 providers to simultaneously refuse retrieval before a data owner is blocked. The 20% ASN cap limits any single correlated group to approximately 11 providers — categorically insufficient for an effective denial attack.

---

## 16. Reliability Scoring

Each provider has a reliability score computed from three rolling audit windows: the last 24 hours (highest weight), the last 7 days, and the last 30 days. The score is the weighted average audit pass rate across the three windows. ([ADR-008](../decisions/ADR-008-reliability-scoring.md))

The score drives two things:
- **Assignment priority.** Higher-scored providers receive more new chunk assignments (Power of Two Choices selection — two candidates drawn randomly, higher scorer wins). No provider is ever guaranteed all assignments; any vetted provider has non-zero selection probability.
- **Payment release.** The 30-day score determines the monthly release multiplier.

The score cannot be gamed by peer-to-peer reputation. The microservice is the sole issuer and verifier of all audit challenges. If providers rated each other instead, the unique Nash equilibrium would be universal dishonesty — every provider would rate every other provider as passing, regardless of actual behaviour. The centralised auditor is not a compromise; it is the only architecture that produces honest outcomes. (Research Papers 37 and 24)

New providers start with an optimistic neutral score and accumulate history through the vetting period.

---

## 17. The Minimum Viable Network

The system will not accept uploads until all of the following conditions are simultaneously true. ([ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md))

| Condition | Threshold |
|---|---|
| Active vetted providers | ≥ 56 |
| Distinct ASNs in active pool | ≥ 5 |
| Distinct Indian metro regions | ≥ 3 |
| Microservice cluster state | Full (3,2,2) quorum |
| Razorpay Linked Accounts | ≥ 56 providers with 24-hour cooling complete |
| Relay infrastructure | ≥ 3 relay nodes deployed |
| Cluster audit secret | Loaded on all replicas |

The assignment service exposes a health endpoint (`GET /api/v1/admin/readiness`) that returns the current state of all conditions. The service re-evaluates every 60 seconds. Upload requests return HTTP 503 until all conditions are met.

The 56-provider minimum is a hard constraint from RS(16,56) — a file needs exactly 56 distinct shard holders. The 5-ASN minimum is a hard constraint from the 20% cap — with fewer than 5 ASNs, one ASN necessarily holds more than 20%.

---

## 18. Consistency Model Summary

For completeness, the full I-confluence map of the system's core operations: ([ADR-013](../decisions/ADR-013-consistency-model.md))

**Coordination-free (14 operations):** All audit receipt inserts, reliability score increments, provider registration (using UUIDv7), file record inserts, chunk record inserts, all reads, escrow deposits, soft deletes, repair triggers, repair completion records, audit challenge issuance, repair scheduling.

**Coordinated (6 operations):** Reliability score decrement (floor ≥ 0), escrow balance decrement on payment release (floor ≥ 0), escrow seizure (same floor), chunk assignment (uniqueness constraint), capability token validation (expiry is time-dependent), physical provider deletion (prohibited — soft delete only).

---

## 19. Known Limitations and V3 Scope

These are not bugs — they are explicit design decisions. Each has a documented rationale.

**No mobile providers.** The repair bandwidth at mobile MTTF (~30–90 days) exceeds the 100 Kbps background budget. Including mobile would make the network spend more bandwidth on repair than on serving actual data. ([ADR-010](../decisions/ADR-010-desktop-only-v2.md))

**No public audit verification.** In V2, a data owner must trust the microservice's countersignature. The V3 Transparent Merkle Log will allow independent verification without trusting the operator. ([ADR-015](../decisions/ADR-015-audit-trail.md))

**India-only at launch.** All payment processing is through Razorpay/UPI. The `PaymentProvider` interface is designed to add Stripe or other gateways later without rewriting payment logic.

**No upload optimality threshold.** Storj uses a parameter `o` to cancel the slowest uploads once enough fragments are confirmed, reducing P99 upload latency. This is a V3 optimisation.

**Repair bandwidth optimisation.** At V2 scale (hundreds of providers), the 39 Kbps/peer bandwidth is manageable. At V3 scale (thousands of providers), Hitchhiker codes — which reduce repair bandwidth by 25–45% with minimal implementation complexity — are the primary candidate. The adoption gate is whether observed V2 bandwidth exceeds 60 Kbps/peer over the first six months. ([ADR-026](../decisions/ADR-026-repair-bw-optimisation.md))

---

## 20. Component Index

Each major component maps to one or more ADRs. Use this table to find the specification for any component:

| Component | ADRs |
|---|---|
| Coordination architecture | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| Proof of storage / audit | [ADR-002](../decisions/ADR-002-proof-of-storage.md) |
| Erasure coding parameters | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Repair protocol | [ADR-004](../decisions/ADR-004-repair-protocol.md) |
| Peer selection and vetting | [ADR-005](../decisions/ADR-005-peer-selection.md) |
| Polling and departure threshold | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Provider exit states | [ADR-007](../decisions/ADR-007-provider-exit-states.md) |
| Reliability scoring | [ADR-008](../decisions/ADR-008-reliability-scoring.md) |
| Background CPU budget | [ADR-009](../decisions/ADR-009-background-execution.md) |
| No mobile in V2 | [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| Fiat escrow / Razorpay | [ADR-011](../decisions/ADR-011-escrow-payments.md) |
| Payment per audit passed | [ADR-012](../decisions/ADR-012-payment-basis.md) |
| Consistency model | [ADR-013](../decisions/ADR-013-consistency-model.md) |
| Adversarial defences | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Audit trail | [ADR-015](../decisions/ADR-015-audit-trail.md) |
| Payment DB schema | [ADR-016](../decisions/ADR-016-payment-db-schema.md) |
| Audit receipt schema | [ADR-017](../decisions/ADR-017-audit-receipt-schema.md) |
| Hot/cold storage bands | [ADR-018](../decisions/ADR-018-hot-cold-storage-bands.md) |
| Client-side cipher | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| Key management | [ADR-020](../decisions/ADR-020-key-management.md) |
| P2P transfer (libp2p + QUIC) | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Encoding pipeline (AONT-RS) | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| Provider storage engine | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| Economic mechanism | [ADR-024](../decisions/ADR-024-economic-mechanism.md) |
| Microservice cluster quorum | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) |
| Repair bandwidth optimisation (V3) | [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) |
| Cluster audit secret | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |
| Provider heartbeat | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| Bootstrap conditions | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |

---

## 21. Critique — Self-Applied

Before finalising this document, the following questions were used to stress-test it:

**Does the document say what builds first?** No — that is intentional. Build sequencing belongs in a separate implementation plan, not in the architecture document. The architecture describes what the system is, not in what order to construct it.

**Is the 20% ASN cap enforceable at launch with only 56 providers across 5 ASNs?** Yes, but only just. At 56 providers and 5 ASNs, the cap allows a maximum of 11 providers per ASN. With 56 providers across 5 ASNs, the natural distribution is about 11 per ASN — the cap is tight from day one. This is why the minimum network condition requires at least 5 ASNs, not 3 or 4.

**What happens during a microservice outage?** P2P chunk retrieval continues without interruption (data owners hold pointer files and can dial providers directly). Audit challenges stop issuing. DHT records begin expiring after 24 hours. New uploads, provider registration, payment releases, and repair queuing all block until the microservice recovers. No data is lost during a microservice outage.

**Is the audit scale feasible?** Full daily audits for 100,000 providers × 10,000 chunks each approaches the ceiling of what a single optimised Postgres instance can handle (~10,000 inserts/second). V2 launches at hundreds of providers — nowhere near this limit. The ceiling exists and is documented. When it is approached, probabilistic sampling must be evaluated against SHELBY Theorem 1 (the incentive compatibility conditions must be re-verified at the new audit frequency).

**Does the pointer file backup story hold?** The microservice stores the encrypted pointer file ciphertext. The data owner's key to decrypt it is never held by the microservice. On device loss with passphrase known: full recovery from the microservice-stored ciphertext. On device loss with no passphrase: recovery requires the BIP-39 mnemonic. On loss of everything: permanent data loss. This must be disclosed clearly at onboarding — it is a non-negotiable property of zero-knowledge storage.