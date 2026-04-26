# Vyomanaut V2 — System Architecture Document

**Version:** 1.0  
**Status:** Design-complete. Build phase beginning.  
**Authors:** Vyomanaut Engineering  
**Last updated:** April 2026  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
**Decisions index:** `docs/decisions/README.md`  
**Research index:** `docs/research/reading-list.md`

---

## Document Purpose and Audience

This document describes the architecture of Vyomanaut V2. It answers three questions a new engineer needs before writing a single line of code: what the system is, how its parts connect, and why each design choice was made.

Every significant decision in this document traces to an Architecture Decision Record (ADR) in `docs/decisions/`. Where this document and an ADR conflict, the ADR wins — it was written with more context. Raise any conflict before building.

This document is for:
- **Engineers building the system** — to understand what they are building and where their component fits.
- **Engineers reviewing code** — to check a proposed change against the architecture.
- **Future engineers joining the project** — to get up to speed without reading all 41 research papers.

This document is not a build plan. It does not say what to build first or how long things take. It describes what the finished system looks like.

---

## Table of Contents

1. [What Vyomanaut Does](#1-what-vyomanaut-does)
2. [What V2 Does Not Do](#2-what-v2-does-not-do)
3. [Quality Attributes](#3-quality-attributes)
4. [Technology Stack](#4-technology-stack)
5. [System Context](#5-system-context)
6. [Component Overview](#6-component-overview)
7. [Trust Boundaries](#7-trust-boundaries)
8. [Deployment Topology](#8-deployment-topology)
9. [Core Design Principles](#9-core-design-principles)
10. [Data Encoding Pipeline](#10-data-encoding-pipeline)
11. [Key Hierarchy](#11-key-hierarchy)
12. [Provider Lifecycle](#12-provider-lifecycle)
13. [P2P Transfer Layer](#13-p2p-transfer-layer)
14. [Audit System](#14-audit-system)
15. [Repair System](#15-repair-system)
16. [Provider Storage Engine](#16-provider-storage-engine)
17. [Payment System](#17-payment-system)
18. [Coordination Microservice](#18-coordination-microservice)
19. [Reliability Scoring](#19-reliability-scoring)
20. [Adversarial Defences](#20-adversarial-defences)
21. [Consistency Model](#21-consistency-model)
22. [Runtime Flows](#22-runtime-flows)
23. [Error Handling](#23-error-handling)
24. [Observability](#24-observability)
25. [Minimum Viable Network](#25-minimum-viable-network)
26. [Known Limitations and V3 Scope](#26-known-limitations-and-v3-scope)
27. [ADR Reference Index](#27-adr-reference-index)

---

## 1. What Vyomanaut Does

Vyomanaut is a paid distributed cold storage network. Data owners pay to store files. Storage providers — people running the Vyomanaut daemon on their home desktop or NAS — earn money by keeping those files available and passing regular storage verification challenges.

Three properties define the system:

**The service never sees the data.** Files are encrypted on the data owner's device before anything leaves it. Providers store encrypted pieces. The coordination microservice never touches file contents. No party in the middle can read anything stored.

**Data is split across 56 independent providers.** A file is broken into 56 encrypted fragments. Any 16 of those 56 are enough to reconstruct the original. A provider can go offline, depart, or fail — the file is unaffected as long as enough fragments survive.

**Payment is tied to proof, not trust.** Providers are paid only when they pass a daily audit: a cryptographic challenge where they prove they still hold the data they were assigned. The correct answer cannot be computed without the actual data. You cannot fake a pass.

---

## 2. What V2 Does Not Do

These are explicit design decisions, each with a documented reason. Building around them wastes time.

- **No mobile providers.** Home desktops and NAS devices only. Operating systems on phones kill background processes unpredictably. Mobile introduces repair bandwidth costs the network cannot absorb at the current erasure parameters. Mobile is explicitly deferred to V3. ([ADR-010](../decisions/ADR-010-desktop-only-v2.md))
- **No blockchain.** The three things blockchains provide in competing systems — an immutable audit log, automatic payment triggers on verified proof, and public dispute resolution — are each replicated by specific microservice components without on-chain writes.
- **No deduplication.** Every file upload uses a fresh random encryption key. Files are not deduplicated across data owners. This is a property of the AONT encoding design.
- **No file versioning or in-place updates.** Files are immutable once stored. Updating a file means deleting the old one and uploading a new one.
- **No retrieval market.** Payment is for proving storage presence, not for serving downloads. A provider earns the same whether or not their data is retrieved.
- **India-only at launch.** All payment processing uses Razorpay and UPI, which require an Indian bank account. The payment abstraction layer is designed to add international gateways later without rewriting the payment logic.

---

## 3. Quality Attributes

These are the non-functional requirements that drove architectural decisions. Every tradeoff in the design can be traced back to at least one of these.

### Durability
**Target:** Data loss rate < 10⁻¹⁵ per year per file.  
**How achieved:** Reed-Solomon RS(16, 56) with lazy repair at r0=8. At the target MTTF of 300 days per provider, the Giroire formula gives an actual loss rate of approximately 10⁻²⁵ per year — four orders of magnitude below the target. The 20% ASN cap is a co-requisite: without it, correlated failures invalidate the calculation. ([ADR-003](../decisions/ADR-003-erasure-coding.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md))

### Availability
**Target:** File retrievable at any time as long as 16 of 56 fragment holders are reachable.  
**How achieved:** 40 parity fragments above the reconstruction threshold. A file is accessible even if 40 providers simultaneously fail. At the target MTTF, the probability of 40 simultaneous failures is negligible.

### Repair bandwidth
**Target:** ≤ 100 Kbps per provider of background upload bandwidth.  
**Actual:** ~39 Kbps per provider at MTTF = 300 days.  
**How achieved:** Lazy repair defers reconstruction until redundancy drops to r0=8 above the reconstruction floor. This produces approximately 38× lower bandwidth than reacting to every failure immediately. ([ADR-004](../decisions/ADR-004-repair-protocol.md))

### Audit response latency
**Target:** Each provider responds to an audit challenge within `(chunk_size / p95_measured_upload_speed) × 1.5`.  
**Typical value:** ~614 ms for a provider with 5 Mbps declared upload throughput.  
**Why it matters:** This deadline makes just-in-time retrieval attacks infeasible — a provider cannot fetch the data from somewhere else and respond in time. ([ADR-014](../decisions/ADR-014-adversarial-defences.md))

### Microservice availability
**Target:** No single microservice replica failure interrupts service.  
**How achieved:** Three replicas with a (3, 2, 2) quorum. One replica can fail; reads and writes continue with the remaining two. ([ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md))

### Provider background CPU usage
**Target:** ≤ 5% of CPU and background I/O from the provider daemon.  
**How achieved:** WiscKey storage engine keeps write amplification at ~1.0 for 256 KB chunks. ChaCha20 AONT encoding completes in ~186 ms per 14 MB segment on hardware without AES acceleration. ([ADR-009](../decisions/ADR-009-background-execution.md), [ADR-023](../decisions/ADR-023-provider-storage-engine.md))

### Zero-knowledge storage
**Target:** The service and its operators must never be able to read any stored data.  
**How achieved:** All encryption happens on the data owner's device before upload. The microservice stores only the encrypted ciphertext of pointer files. Providers store only encrypted fragments. No party in the coordination or audit path holds a decryption key. ([ADR-019](../decisions/ADR-019-client-side-encryption.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md))

---

## 4. Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Microservice language | Go | Concurrent, low-overhead, good libp2p support |
| Provider daemon language | Go | Same binary can support future cross-platform builds |
| Microservice database | PostgreSQL | Append-only INSERT-only audit log; CRDT-compatible escrow ledger; row security policy enforcement |
| Provider storage engine | RocksDB (index) + custom append-only vLog (values) | WiscKey key-value separation; write amplification ~1.0 at 256 KB |
| P2P networking | libp2p | Production-deployed in IPFS and Filecoin; full NAT traversal stack; Kademlia DHT included |
| Primary transport | QUIC v1 (RFC 9000) | Connection migration survives IP changes; independent stream delivery; built-in TLS 1.3 |
| Fallback transport | TCP + Noise XX + yamux | Identical hole-punch success rate to QUIC; activates when UDP is blocked |
| Client-side encryption | ChaCha20-256 (no AES-NI) / AES-256-CTR (AES-NI) | ChaCha20 is constant-time without hardware; 3× faster than AES on low-end desktops |
| Authenticated encryption | AEAD_CHACHA20_POLY1305 | For pointer file encryption; RFC 8439 standard |
| Erasure coding | Reed-Solomon RS(16, 56) | Only MDS code family that supports arbitrary (n, k) at our parameters |
| Key derivation | HKDF-SHA256 and Argon2id | HKDF for operational key derivation; Argon2id for master secret from passphrase |
| Digital signatures | Ed25519 | Sub-millisecond key generation; used for audit receipts and pointer file integrity |
| Payment gateway | Razorpay (Route + Smart Collect 2.0 + RazorpayX Payouts) | India-first; UPI; no per-transaction fee for P2P transfers |
| Secrets management | HashiCorp Vault / AWS SSM / GCP Secret Manager | For cluster audit secret distribution across microservice replicas |

---

## 5. System Context

Vyomanaut sits between data owners who need storage and providers who have idle disk space. The coordination microservice is the control plane. The P2P network is the data plane. These two planes are deliberately separate — the microservice is never in the path of actual file data.

```
External parties:
  DATA OWNER ──────────────────────────────────────────────── PROVIDER (×56+)
  (desktop app / web UI)                                      (daemon on home desktop / NAS)
       │                                                              │
       │ HTTPS: deposit, upload metadata, retrieve pointer            │ libp2p/QUIC: chunk upload/download
       │ QUIC: direct P2P chunk upload (to providers directly)        │ HTTPS: heartbeat, audit response
       │                                                              │
       └────────────────── COORDINATION MICROSERVICE ────────────────┘
                           (control plane only — never data)
                                       │
                            ┌──────────┴──────────┐
                            │                     │
                       RAZORPAY               SECRETS MANAGER
                    (payment rails)         (cluster audit secret)
```

What the microservice knows: which chunk is on which provider, who passed their last audit, who should be paid, and the encrypted ciphertext of pointer files (which it cannot decrypt).

What the microservice never touches: file plaintext, AONT keys, decryption keys of any kind.

---

## 6. Component Overview

The system has six first-class components and three external dependencies.

**First-class components:**

| Component | Runs on | Primary responsibility |
|---|---|---|
| Coordination microservice | Cloud VMs (3 replicas) | Registration, audit scheduling, payment, repair orchestration |
| Provider daemon | Provider desktop / NAS | Chunk storage, audit response, heartbeat |
| Data owner client | Data owner's device | Encryption, encoding, upload, retrieval |
| Relay nodes | Cloud VMs (3 at launch) | NAT traversal fallback for symmetric-NAT providers |
| Kademlia DHT | Distributed across all provider daemons | Chunk-address lookup (data plane routing) |
| PostgreSQL cluster | Co-located with microservice | Audit log, escrow ledger, chunk assignment table, provider registry |

**External dependencies:**

| Dependency | Purpose | Failure impact |
|---|---|---|
| Razorpay Route + Smart Collect 2.0 | Provider payment releases and data owner deposits | Payment releases pause; audits and storage continue unaffected |
| Secrets manager | Cluster audit secret distribution | Microservice replicas cannot start; existing instances continue running with cached secret |
| Indian ISP infrastructure | Connectivity between providers and data owners | Covered by 40-fragment parity and 3-region relay deployment |

### Component graph

The diagram below shows all six first-class components with the protocol on every connection. HOT = on the audit cycle (latency-sensitive); COLD = administrative or background.

```
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  DATA OWNER CLIENT (desktop / web)                                           │
  └────┬────────────────────────────────────────┬───────────────────────────────┘
       │                                         │
       │ HTTPS  POST /upload/assign [COLD]        │ libp2p/QUIC  CHUNK_REQUEST /
       │ HTTPS  GET  /file/*        [COLD]        │             CHUNK_RESPONSE [HOT]
       │                                         │
  ┌────▼──────────────────────────┐         ┌───▼─────────────────────────────────┐
  │  COORDINATION MICROSERVICE     │         │  PROVIDER DAEMON (×56+)             │
  │  (3 replicas, gossip cluster)  │         │  WiscKey storage engine             │
  │                                │◄────────┤                                     │
  │  [control plane only]          │ HTTPS   │  libp2p heartbeat → microservice    │
  │                                │ HTTPS   │  HTTPS  audit challenge response     │
  └────┬──────────┬────────────────┘ [HOT]   └───┬────────────────┬───────────────┘
       │          │                               │                │
       │          │ SQL  read/write [HOT]          │ libp2p/QUIC   │ libp2p/QUIC
       │          │                               │ DHT FIND_VALUE │ repair transfer
       │     ┌────▼──────────────┐                │ [COLD]         │ [COLD]
       │     │  POSTGRESQL        │           ┌────▼───────────────▼──────────────┐
       │     │  audit_receipts    │           │  KADEMLIA DHT                      │
       │     │  escrow_events     │           │  (runs inside each provider daemon)│
       │     │  providers         │           │  chunk_id (HMAC) → provider addr   │
       │     │  chunk_assignments │           └────────────────────────────────────┘
       │     └───────────────────┘
       │
       │ libp2p  QUIC dial (challenge dispatch) [HOT]
       │ libp2p  Circuit Relay v2 coordination  [HOT]
       │
  ┌────▼──────────────────────────┐
  │  RELAY NODES (3 at launch)     │
  │  one per availability zone     │
  │  128 concurrent reservations   │
  │  each; fallback for sym. NAT   │
  └────────────────────────────────┘

Edge legend:
  [HOT]  = on the daily audit cycle; latency-sensitive; must meet per-provider RTO
  [COLD] = administrative, background, or infrequent; no latency SLA
  All data plane transfers (chunk upload/download) bypass the microservice entirely.
```

---

## 7. Trust Boundaries

This table defines what each component is and is not trusted to do. It determines where verification must happen rather than trust.

| Component | Trusted for | Not trusted for |
|---|---|---|
| Coordination microservice | Audit scheduling, payment computation, chunk assignment, audit result recording | Reading file contents; holding decryption keys; self-reporting its own correctness (V3 Merkle Log addresses this) |
| Provider daemon | Storing the chunk it was assigned and responding to challenges | Reporting its own reliability; self-certifying throughput; knowing what other providers hold |
| Data owner client | Encrypting correctly before upload; holding the pointer file | Providing correct audit results; knowing which providers hold which fragments |
| Relay nodes | Forwarding QUIC/TCP traffic for NAT traversal | Reading packet contents (all traffic is TLS 1.3 encrypted end-to-end) |
| Razorpay | Processing payment transfers per API calls | Computing payout amounts (microservice does this and sends exact paise amounts) |
| PostgreSQL | Enforcing INSERT-only policy on audit_receipts (row security policy) | Validating audit response correctness (microservice verifies before inserting) |

**Critical trust boundary: the AONT threshold.** Security depends on no single party holding k=16 or more fragments of the same file. The 20% ASN cap keeps any correlated group below 12 of 56 fragments. If the cap is not enforced at assignment time, the zero-knowledge property degrades.

---

## 8. Deployment Topology

**Cloud provider.** AWS or GCP — operator's choice at deployment time. Both are acceptable. The architecture has no dependency on cloud-provider-specific features; managed Postgres (RDS or Cloud SQL) and standard VM instances are the only cloud primitives used. The remainder of this section gives AWS names in parentheses as a concrete reference; substitute GCP equivalents if deploying there.

**Coordination microservice.** Three VM instances (e.g. AWS EC2 `t3.medium` or equivalent: **2 vCPU, 4 GB RAM**), one per availability zone in `ap-south-1` (Mumbai). Each runs the microservice binary. They share a managed Postgres primary in the same region with two read replicas across AZs. Gossip membership connects all three replicas directly. External administrative traffic routes through a load balancer; audit challenge dispatch and assignment calls bypass the load balancer via client-driven direct routing.

**Postgres.** Managed Postgres with Multi-AZ replication enabled (e.g. AWS RDS `db.t3.medium`: **2 vCPU, 4 GB RAM**, 100 GB GP3 SSD to start, auto-scaling enabled). The microservice writes to the primary; read-heavy paths (score queries, assignment lookups) use a read replica. Monthly partition archival of `audit_receipts` is mandatory — partitions older than 30 days move to object storage (S3 / GCS).

**Relay nodes.** Three instances (**1 vCPU, 1 GB RAM, minimum 1 Gbps network**), one per availability zone. At launch: Delhi AZ, Mumbai AZ1, Mumbai AZ2 (or Chennai as third). Each runs libp2p with Circuit Relay v2 enabled, sized for 128 concurrent relay reservations — 384 total slots, 4.3× headroom at 300 initial providers. Scale to a fourth node when provider count exceeds 570 (the relay saturation point at 45% CGNAT fraction).

**Provider daemon.** Runs on provider hardware — home desktop or NAS. Cross-platform binary for Windows 10+, macOS 12+, and Ubuntu 22.04+. No cloud dependency. Connects outbound to the microservice via HTTPS and to other providers via libp2p.

**Data owner client.** Desktop app or web application. Runs the encoding pipeline locally (ChaCha20 AONT + Reed-Solomon) before any data leaves the device.

**Secrets manager.** A managed secrets service (HashiCorp Vault, AWS SSM Parameter Store, or GCP Secret Manager) accessible by all three microservice replicas. Stores the cluster master seed under versioned paths (`/vyomanaut/audit-secret/v{N}`). Not in the hot path — loaded at replica startup and cached for 5 minutes. If the secrets manager is unreachable at startup, the replica does not start (fail-closed).

---

## 9. Core Design Principles

These seven principles governed every architectural decision. When a new engineering choice comes up during the build, check it against these before deciding.

**Lazy everything.** Work is deferred until necessary. Repair fires only when fragment count drops to a threshold, not immediately on every departure. This single decision reduces repair bandwidth by ~38× compared to eager repair. ([ADR-004](../decisions/ADR-004-repair-protocol.md))

**Prove, don't trust.** No component self-reports its own state. Storage is verified by cryptographic challenge-response. Throughput is measured during audits, not taken from declarations. The microservice issues all challenges; providers cannot influence when they are tested. ([ADR-002](../decisions/ADR-002-proof-of-storage.md))

**Coordinate only where necessary.** 14 of 20 core operations scale horizontally without any coordination. Only 6 require a single authoritative source. The system is designed to push as many operations as possible into the coordination-free category. ([ADR-013](../decisions/ADR-013-consistency-model.md))

**Bound correlated failures structurally.** When providers share an ISP, city, or power grid, they fail together. The system prevents this from causing data loss by ensuring no single correlated group holds more than 20% of any file's fragments at assignment time. This cap is both a security requirement and a durability requirement. ([ADR-014](../decisions/ADR-014-adversarial-defences.md))

**Pay for presence, not transfer.** Providers earn per audit passed, not per gigabyte transferred. This keeps the payment system and the P2P transfer layer independent — a payment outage cannot accumulate credit liability, and a transfer outage does not stop audits. ([ADR-012](../decisions/ADR-012-payment-basis.md))

**Fail closed on cryptographic operations.** If the cluster audit secret cannot be loaded, a microservice replica does not start. If a chunk's content hash fails verification at read time, the audit result is FAIL — never a wrong hash. Unknown is always worse than a known failure.

**Data plane and control plane are separate.** File data never flows through the microservice. The microservice knows about data (which chunk is where, who holds it) but never holds the data itself. This separation is not a performance optimisation — it is the mechanism by which zero-knowledge storage holds even if the microservice is compromised.

---

## 9.5 Security Boundary Summary

The table below answers "can Vyomanaut staff read my files?" for every plausible attack path. It is the single reference for security reasoning during code review.

| Threat | What limits it | ADR |
|---|---|---|
| Operator or microservice reads stored files | The AONT key K is never transmitted to or stored by the microservice. K is recoverable only by assembling k=16 fragments from providers. The microservice holds only encrypted pointer file ciphertext it cannot decrypt. | [ADR-019](../decisions/ADR-019-client-side-encryption.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| Compromised provider reads files it does not hold | A provider holds one of 56 fragments. Decrypting any word requires K. K requires all s+1 AONT codewords. A single fragment reveals nothing. | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| Compromised provider reads files it does hold (k=16 threshold breach) | The 20% ASN cap ensures no correlated group holds more than ~11 of 56 fragments. Collusion at or below 11 providers cannot reach the k=16 threshold. | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| DHT observer correlates lookup traffic to file identity | DHT lookup keys are `HMAC-SHA256(chunk_hash, file_owner_key)`. The file_owner_key is derived from the data owner's master secret. No observer can map a DHT key to a file without the owner's credentials. | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| Replay of a valid audit response from a previous challenge | The challenge nonce is `HMAC(server_secret, chunk_id + server_ts)`. A nonce is used exactly once (unique index on `challenge_nonce` in `audit_receipts`). Replaying an old nonce collides with the unique constraint. | [ADR-017](../decisions/ADR-017-audit-receipt-schema.md), [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |
| Silent disk corruption producing a wrong but accepted audit response | The provider daemon verifies `SHA-256(chunk_data) == content_hash` from the vLog entry before computing any response. If verification fails, the result is `FAIL` — a wrong hash is never returned to the microservice. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| Provider outsources data retrieval just before a challenge | The audit deadline `(chunk_size / p95_throughput) × 1.5` is shorter than the round-trip time to fetch 256 KB from another provider over a residential connection. JIT retrieval is physically infeasible within the window. | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Cluster secret leaks and nonces become predictable | Nonces include a version byte; rotation retires old secrets after 24 hours. The master seed lives only in the secrets manager, never on disk in any replica. | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |

---

## 10. Data Encoding Pipeline

Before any data leaves the data owner's device, it passes through a four-stage pipeline that transforms it into 56 encrypted, independent fragments.

### Segmentation

Files larger than 14 MB are split into multiple segments before encoding. Each segment is at most 14 MB (56 × 256 KB). Every segment is processed independently through Stages 1–4 below. The pointer file contains one entry per segment, including the provider list and chunk IDs for that segment's 56 fragments. Retrieval reconstructs each segment independently and concatenates them in order to rebuild the original file. There is no cross-segment state — losing all 56 fragments of one segment does not affect any other segment's recoverability.

### Stage 1 — AONT encryption

A fresh random 256-bit key K is generated for the segment. The AONT (All-or-Nothing Transform) processes the segment as follows:

1. Append a fixed-value 16-byte canary word to the segment.
2. Generate K = SecureRandom(256 bits).
3. For each 16-byte word d_i in the segment: `c_i = d_i XOR cipher(K, counter=i)`.
4. Compute the commitment hash: `h = SHA-256(c_0 || c_1 || ... || c_s)`.
5. Append the key-embedding block: `c_{s+1} = K XOR h`.

The security property: recovering K requires computing h, which requires all s+1 codewords. An attacker with fewer than k=16 fragments cannot assemble the full package, cannot compute h, cannot recover K, and cannot decrypt any word. The key is embedded in the data and protected by the erasure threshold. ([ADR-022](../decisions/ADR-022-encryption-erasure-order.md))

The cipher is ChaCha20-256 on hardware without AES-NI (the common case for Indian desktops), and AES-256-CTR on hardware with AES-NI. The daemon detects this at startup via CPUID and sets a global constant. Both paths produce identical outputs from the AONT's perspective — only the performance differs. ([ADR-019](../decisions/ADR-019-client-side-encryption.md))

### Stage 3 — Reed-Solomon erasure coding

The AONT package feeds into systematic Reed-Solomon coding with parameters s=16, r=40, producing 56 fragments of 256 KB each. The first 16 fragments are the direct AONT package words (the systematic property). The remaining 40 are parity.

The parameters come from the Giroire optimality condition: r=40 is the unique value that minimises repair bandwidth at s=16, r0=8. Choosing any other r increases bandwidth. ([ADR-003](../decisions/ADR-003-erasure-coding.md))

### Stage 4 — Upload and pointer file creation

The 56 fragments upload directly to 56 providers selected by the microservice's assignment service. On completion, the data owner's client creates the pointer file containing the 56 provider IDs, 56 chunk content addresses (SHA-256 of each fragment), and the erasure parameters. The pointer file is encrypted with AEAD_CHACHA20_POLY1305 before storage. The microservice stores the encrypted ciphertext and cannot decrypt it. ([ADR-020](../decisions/ADR-020-key-management.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md))

### Decoding

To retrieve a file, the data owner contacts any 16 of the 56 providers, downloads their fragments, runs Reed-Solomon decode to recover the AONT package, recovers K from the package (h is computable from the full package, K = c_{s+1} XOR h), decrypts each word, and verifies the canary. If the canary is wrong, the segment is corrupt — escalate to the audit subsystem.

---

## 11. Key Hierarchy

The data owner's credentials derive from a single master secret that is never written to disk.

```
passphrase + owner_id
      │
      ▼  Argon2id (t=3, m=64 MB, p=4)
  master_secret  ─────────────────────────────── BIP-39 mnemonic (offline backup)
      │
      ├─ HKDF("vyomanaut-file-v1" || file_id)       ──► file_key  (per file, for future use)
      │
      ├─ HKDF("vyomanaut-pointer-v1" || file_id)    ──► pointer file encryption key
      │                                                       │
      │                                             AEAD_CHACHA20_POLY1305 encrypt
      │                                                       │
      │                                             encrypted pointer file
      │                                             (stored by microservice, unreadable to it)
      │
      └─ HKDF("vyomanaut-keystore-v1")              ──► key store encryption key
                                                            │
                                                    encrypts: Ed25519 signing key
                                                              + pointer file nonce counter
```

The AONT key K is not in this hierarchy. It is embedded in the erasure-coded fragments and recovered automatically when k=16 fragments are assembled. The data owner never manages K directly.

Recovery paths:
- **Device loss, passphrase known:** Re-derive master_secret; download and decrypt pointer file ciphertext from microservice. Full recovery.
- **Device loss, no passphrase, BIP-39 mnemonic available:** Reconstruct master_secret from mnemonic. Same as above.
- **All credentials lost:** Permanent, unrecoverable data loss. This must be clearly disclosed at onboarding.

([ADR-020](../decisions/ADR-020-key-management.md))

---

## 12. Provider Lifecycle

### Joining

**Registration.** The provider installs the daemon, which generates an Ed25519 key pair on first run. The provider registers via the app with their phone number (OTP-verified) and public key. The microservice creates a provider record (`status = PENDING_VERIFICATION`) and initiates a Razorpay Route Linked Account creation. A 24-hour cooling period must pass before the first payment transfer. ([ADR-001](../decisions/ADR-001-coordination-architecture.md))

**First heartbeat.** Once running, the daemon sends a signed heartbeat to the microservice control plane every 4 hours, reporting the provider's current libp2p multiaddresses. Indian residential ISPs frequently rotate DHCP leases — the heartbeat ensures the microservice always has a fresh address for audit challenge dispatch. Status advances to `VETTING`. ([ADR-028](../decisions/ADR-028-provider-heartbeat.md))

**Vetting period (4–6 months).** The provider receives non-critical chunk assignments under extra erasure redundancy while the system measures their reliability. Earnings accumulate under a 60-day hold window with a 50% release cap. After 80+ consecutive audit passes — the statistical threshold where a 99% confidence estimate of reliability becomes achievable — the provider advances to `ACTIVE`. ([ADR-005](../decisions/ADR-005-peer-selection.md), [ADR-024](../decisions/ADR-024-economic-mechanism.md))

**Full operation.** The hold window shortens to 30 days. The release cap is removed. The provider competes for new chunk assignments alongside all other active providers. Higher reliability scores win more assignments. ([ADR-008](../decisions/ADR-008-reliability-scoring.md))

### During operation

The provider daemon continuously:
- Sends a heartbeat every 4 hours to the microservice with its current network addresses
- Receives audit challenge messages daily for each stored chunk
- Responds to challenges within the per-provider deadline
- Receives new chunk assignments via the assignment service
- Stores new chunks in the WiscKey engine on local disk
- Accumulates earnings in the microservice's internal escrow ledger

### Exiting

Four exit states with distinct triggers and financial consequences. ([ADR-007](../decisions/ADR-007-provider-exit-states.md))

| State | Condition | Repair triggered | Escrow outcome |
|---|---|---|---|
| Temporary absence | Absent < 72 h, no notice | No | Score decremented per polling cycle |
| Promised downtime | Absence declared in advance | No — wait for promised period | Fine deducted if promise broken |
| Silent departure | Absent ≥ 72 h, no announcement | Immediately | All held escrow seized for repair fund |
| Announced departure | Provider explicitly notified | Immediately | Held escrow released proportionally |

The 72-hour threshold is derived from Bolosky's measured desktop absence distribution: 99.7% of weekend absences resolve within 70 hours. The threshold is set above the peak normal absence to avoid false-positive repair triggers.

**Returning after silent departure.** When a provider is declared silently departed, the microservice sets `status = DEPARTED`, removes all their chunk assignments from the assignment table (stopping further challenge issuance), seizes their escrow, and queues repair for all affected chunks. If the provider later comes back online, the microservice returns HTTP 403 on all requests. Re-joining requires a full registration with a new provider_id. Prior chunk data on disk is not re-integrated — it has already been replaced by repair.

---

## 13. P2P Transfer Layer

File data moves directly between data owner and providers, and between providers during repair. The microservice is never in this path. ([ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md))

### Framework

libp2p handles peer identity, connection establishment, NAT traversal, stream multiplexing, and the Kademlia DHT. It is the same stack used in production by IPFS and Filecoin.

### Transports

**Primary: QUIC v1 (RFC 9000).** Each chunk transfer is one independent QUIC stream on an existing connection. A lost packet on one stream does not stall others. QUIC connections are identified by a Connection ID rather than an IP:port pair — when a provider's IP changes due to DHCP rotation, in-flight transfers survive automatically. TLS 1.3 is built into QUIC; no separate security handshake is needed.

**Fallback: TCP + Noise XX + yamux.** Activates automatically when a provider is behind a UDP-blocking middlebox. Empirically, TCP and QUIC achieve statistically identical NAT hole-punch success rates (~70%) — the fallback carries no reliability penalty. yamux provides the same independent stream semantics as QUIC over the TCP connection.

### NAT traversal — three tiers

| Tier | Protocol | Applies to | Notes |
|---|---|---|---|
| 1 | AutoNAT | All providers at connect | Classifies: publicly reachable / cone NAT (punchable) / symmetric NAT (relay required) |
| 2 | DCUtR (Direct Connection Upgrade through Relay) | Cone NAT — common on Indian home routers | Relay coordinates simultaneous dial from both sides. 97.6% of successes happen on the first attempt. Retry count set to 1 (not the libp2p default of 3). |
| 3 | Circuit Relay v2 | Symmetric NAT — approximately 30% of providers | All traffic routed through a Vyomanaut-operated relay. Relay overhead < 50 ms for Indian cloud-hosted relays; fits within audit deadline. |

**Relay infrastructure at launch:** Three relay nodes, one per Indian cloud availability zone. Each handles 128 concurrent relay reservations — 384 total slots, 4.3× headroom at 300 initial providers. ([Q30-1, answered](../research/answered-questions.md))

### Peer identity

Each provider daemon generates an Ed25519 key pair at installation. The libp2p Peer ID is `multihash(public_key)`. Every transport connection authenticates the remote Peer ID cryptographically during the handshake — providers cannot impersonate each other.

### DHT configuration

The Kademlia DHT handles chunk-address lookup (finding which provider holds which fragment). It does not handle provider discovery — that goes through the microservice.

| Parameter | Value | Reason |
|---|---|---|
| k-bucket size | 16 | S/Kademlia disjoint-path design: k = 2×d where d=8 |
| Parallel lookups (alpha) | 3 | Produces O(log n / 3) round trips; sub-second median lookup |
| DHT mode for providers | Server (full participant) | Desktop providers must be reachable for challenge dispatch |
| Key namespace | Custom HMAC validator | Prevents file identity leakage in DHT |

DHT lookup keys use: `HMAC-SHA256(chunk_hash, file_owner_key)` where `file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)`. Only the file owner can reverse-map a DHT key to its chunk. The DHT never sees real chunk hashes or file IDs.

DHT records are republished every 12 hours by the microservice's availability service. Record expiry is 24 hours, providing a 12-hour buffer against delayed republication. ([ADR-006](../decisions/ADR-006-polling-interval.md))

### Session resumption policy

0-RTT session resumption (zero round-trip reconnect) is disabled for all connections carrying an audit challenge or signed receipt. 0-RTT data can be replayed by an attacker — unacceptable for operations with payment or audit consequences. 0-RTT may be enabled for pure chunk data transfers where replay has no security consequence.

---

## 14. Audit System

The audit system is how the microservice verifies that providers hold the chunks they are supposed to hold.

### Challenge issuance

Once per day per assigned chunk, the microservice issues an audit challenge containing the chunk ID and a nonce:

```
challenge_nonce = HMAC-SHA256(server_secret, chunk_id + server_timestamp)
```

`server_timestamp` is generated by the microservice — providers cannot influence it. The nonce is versioned (1 prefix byte) so any replica can validate any challenge across replica boundaries. ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md))

The microservice dispatches challenges using the address from `providers.last_known_multiaddrs` (updated by the 4-hour heartbeat), not from the DHT. DHT is used only as a fallback if no heartbeat has been received within 8 hours.

The per-challenge response timeout uses a TCP-style per-provider RTO:

```
RTO = AVG + 4 × VAR
```

where AVG and VAR are the exponentially weighted mean and variance of that provider's recent audit response times. New providers use the pool median. This distinguishes a slow provider (high variance, wait longer) from one that has departed (consistently unresponsive). ([ADR-006](../decisions/ADR-006-polling-interval.md))

### Provider response

The provider receives the challenge and:
1. Looks up the chunk_id in RocksDB. If absent, returns FAIL immediately (no disk I/O needed).
2. Reads the chunk from the vLog using the stored offset.
3. Verifies `SHA-256(chunk_data) == stored_content_hash`. If wrong, returns FAIL — disk corruption detected.
4. Computes `response_hash = SHA-256(chunk_data || challenge_nonce)`.
5. Returns the signed audit receipt.

### Receipt recording

The microservice verifies the response_hash, then writes one row to `audit_receipts`. The table is INSERT-only — no row is ever updated or deleted, enforced by Postgres row security policy. Both the provider and microservice sign the receipt with Ed25519. ([ADR-017](../decisions/ADR-017-audit-receipt-schema.md), [ADR-015](../decisions/ADR-015-audit-trail.md))

### The 12-field receipt schema

| Field | Purpose |
|---|---|
| receipt_id (UUIDv7) | Time-ordered primary key; no coordinator needed |
| schema_version | Forward compatibility |
| chunk_id | Content address of the chunk |
| file_id | Pseudonymous file handle |
| provider_id | Which provider responded |
| challenge_nonce | Versioned nonce; prevents replay |
| server_challenge_ts | Set by server; prevents backdating |
| response_hash | The proof: SHA-256(chunk_data ‖ nonce) |
| response_latency_ms | Just-in-time retrieval detector |
| audit_result | PASS / FAIL / TIMEOUT |
| provider_sig | Ed25519 over all fields above |
| service_sig | Ed25519 over provider_sig + countersign timestamp |

### Crash-safe receipt writing

The microservice uses a two-phase write to survive crashes between challenge validation and receipt countersignature:
1. Write a PENDING row (audit_result = NULL) with the full provider_sig. This is durable before any further processing.
2. Validate the response_hash.
3. Update the row: set audit_result = PASS|FAIL, service_sig, service_countersign_ts.
4. Return the countersignature to the provider.

If the microservice crashes between step 1 and step 3: the provider receives no countersignature. On the next audit cycle (24 hours), a new challenge is issued. The orphaned PENDING row is garbage-collected after 48 hours (marked ABANDONED, not counted in any score window). If the provider retries the same response before the next cycle, the microservice detects the duplicate challenge_nonce and returns the existing receipt.

---

## 15. Repair System

### When repair fires

Repair fires in two situations: a provider is declared silently departed (absent ≥ 72 hours), or the fragment count for a chunk drops to s + r0 = 24 (the lazy repair threshold, 8 above the reconstruction floor of 16).

The 72-hour threshold is calibrated to Bolosky's bimodal desktop absence distribution: nightly absences peak at µ=14h (99.7% return within 20h), weekend absences peak at µ=64h (99.7% return within 70h). 72 hours safely clears the weekend peak without triggering false-positive repair. ([ADR-006](../decisions/ADR-006-polling-interval.md))

### What repair does

The repair scheduler contacts k=16 surviving fragment holders, downloads their fragments, runs Reed-Solomon decode to reconstruct the AONT package, re-encodes the missing fragments, and uploads them to newly selected providers. New providers are selected with the same ASN diversity constraints as original assignments.

The scheduler prioritises confirmed permanent departures (providers past the 72-hour threshold) over pre-warning jobs (chunks approaching r0 but not yet there). This prevents permanently degraded chunks from accumulating behind lower-priority work. ([ADR-039](../decisions/ADR-004-repair-protocol.md))

### Bandwidth budget

For N=1,000 providers, a single provider failure produces approximately 793 GB of total repair network transfer. At 100 Kbps per provider (aggregate 100 Mbps), repair completes in about 8 hours — inside the 12-hour safety window before a second failure becomes likely. The steady-state per-provider repair bandwidth is ~39 Kbps, well inside the 100 Kbps background budget. ([ADR-003](../decisions/ADR-003-erasure-coding.md))

### Emergency floor

Regardless of the 72-hour threshold, if fragment count for any chunk drops to s=16 (the reconstruction floor), repair triggers immediately. No file ever sits at the minimum threshold waiting for a timeout to expire.

---

## 16. Provider Storage Engine

Each provider daemon stores chunks using WiscKey-style key-value separation: the chunk index lives in RocksDB; chunk data lives in a separate append-only value log (vLog). ([ADR-023](../decisions/ADR-023-provider-storage-engine.md))

### Why not standard RocksDB

Standard RocksDB stores keys and values together and rewrites both during compaction. At 256 KB value sizes, this produces 10–14× write amplification. With WiscKey separation, compaction moves only the small 44-byte index entries — values never move. Write amplification drops to approximately 1.0 at 256 KB.

### The chunk index (RocksDB)

Maps `chunk_id (32 bytes) → (vlog_offset uint64, chunk_size uint32) (12 bytes)`. Total entry size: 44 bytes. The index is small enough to stay entirely in RocksDB's block cache after warm-up — audit challenge lookups typically require no disk I/O for the index.

Bloom filters are enabled (10 bits per key, ~1% false-positive rate). An audit challenge for a chunk not assigned to this provider hits only the Bloom filter in memory and returns FAIL instantly.

### The value log (vLog)

A single append-only file on the provider's storage device. Fixed-size entries of 262,212 bytes each:

```
chunk_id        (32 bytes)  — copy for GC validation
chunk_size      (4 bytes)   — always 262,144 in V2
chunk_data      (262,144 bytes) — raw 256 KB fragment
content_hash    (32 bytes)  — SHA-256(chunk_data), verified on every read
```

The `content_hash` is verified on every read before computing the audit response. If disk corruption has altered the chunk, the verification fails and the audit result is FAIL — the microservice is informed, and repair is triggered. Silent disk corruption cannot produce a wrong-but-plausible PoR response.

### Single writer goroutine

All vLog appends are serialised through one writer goroutine. Multiple upload goroutines submit write requests via a buffered channel and block until the writer confirms the vlog_offset. POSIX O_APPEND atomicity does not hold for writes above ~4 KB; for 262 KB entries, serialisation is required.

### Crash recovery

On restart, the daemon reads the last vLog head pointer from RocksDB and scans forward, re-inserting any index entries that were appended to the vLog but not yet flushed to RocksDB. Maximum scan: one memtable flush interval worth of entries, typically a few hundred chunks.

### Audit lookup path

1. Receive `(chunk_id, challenge_nonce)` from the microservice.
2. Check Bloom filter — absent → return FAIL (no disk I/O).
3. Read `(vlog_offset, chunk_size)` from RocksDB block cache (no disk I/O).
4. Read 262,212 bytes from vLog at offset — one random read: ~1 ms SSD / ~12–15 ms HDD.
5. Verify `SHA-256(chunk_data) == content_hash`. Fail immediately on mismatch.
6. Compute `response_hash = SHA-256(chunk_data || challenge_nonce)`.
7. Return signed audit receipt.

Both SSD and HDD times are well inside the audit deadline.

---

## 17. Payment System

### How Razorpay is used

Three Razorpay products, each serving a distinct purpose:

**Smart Collect 2.0** — each data owner is assigned a virtual UPI ID (Customer Identifier). When they deposit funds, Razorpay automates reconciliation and notifies the microservice via webhook. The microservice credits the data owner's escrow account.

**Razorpay Route** — each provider is a Linked Account. After a 24-hour cooling period from account creation, the microservice can create transfers with `on_hold: true`. The Modify Settlement Hold API updates `on_hold_until` on each monthly computation, controlling when earnings are released.

**RazorpayX Payouts** — monthly payment transfers from the master account to providers' registered bank accounts via IMPS or UPI.

Note: **Razorpay Escrow+ is not used.** It requires NBFC registration and a trustee arrangement. Vyomanaut does not qualify. The Route `on_hold` mechanism provides the required partial-hold-and-release behaviour without regulatory preconditions.

**UPI Collect is deprecated as of February 2026.** All data owner deposit flows must use UPI Intent (payer selects their UPI app from a displayed list) or QR code. The Smart Collect 2.0 receiving side is unaffected.

**Idempotency is mandatory.** Every payout API call includes the `X-Payout-Idempotency` header, whose value is `SHA-256(provider_id + audit_period)` taken directly from the `idempotency_key` column in `escrow_events`. Razorpay rejects payout calls without this header as of March 2025.

### Internal escrow ledger

The microservice maintains an append-only `escrow_events` table. Balance is never stored as a column — it is always computed:

```sql
balance =
  SUM(amount_paise) WHERE event_type = 'DEPOSIT'
  - SUM(amount_paise) WHERE event_type IN ('RELEASE', 'SEIZURE')
  for a given provider_id
```

All amounts are stored as integer paise (₹1 = 100 paise). No floating-point arithmetic anywhere in the payment path. ([ADR-016](../decisions/ADR-016-payment-db-schema.md))

### Monthly release computation

On the 23rd of each month, the microservice computes each provider's releasable balance. Razorpay releases on the next business day after `on_hold_until` — the target window for providers is "within the first 3 business days of the following month." The microservice sets `on_hold_until` to the last working day of the current month, accounting for RBI bank holidays via a static lookup table updated each December.

The release multiplier is determined by the provider's 30-day reliability score:

| 30-day score | Release multiplier |
|---|---|
| ≥ 0.95 | 1.00 — full release |
| 0.80–0.94 | 0.75 — partial release |
| 0.65–0.79 | 0.50 — partial release |
| < 0.65 | 0.00 — hold in full |

Withheld portions from partial releases are not seized. They roll into the next month's escrow window. Seizure only occurs on silent departure.

If the 7-day score drops more than 0.20 below the 30-day score (the deterioration signal), the next release uses the lower multiplier of the two score windows — even before the 72-hour departure threshold is crossed. This catches providers who are degrading before they disappear. ([ADR-024](../decisions/ADR-024-economic-mechanism.md))

### Escrow seizure on departure

When a provider is declared silently departed (72 hours without contact):
1. The microservice freezes the provider's account.
2. All earnings in the 30-day rolling window are transferred to the repair reserve fund (SEIZURE event appended to `escrow_events`).
3. A Razorpay Route reversal is issued if any transfer has not yet settled.
4. Repair is triggered for all affected chunks. The seized escrow funds the cost of onboarding replacement providers.

### Vetting period economics

During the 4–6 month vetting period, the hold window is 60 days and the release cap is 50%. A provider in vetting can earn but can access at most 50% of any month's earnings until they pass vetting. This acts as a temporal entry cost without requiring a financial pre-commitment.

---

## 18. Coordination Microservice

### What it does

The microservice is the system's control plane. It handles: provider registration and KYC verification, chunk assignment, audit challenge scheduling and dispatch, audit receipt recording, reliability score maintenance, repair job queuing, payment release computation, DHT record republication, and heartbeat address updates.

It never touches file contents.

### Cluster configuration

Three replicas with a (3, 2, 2) quorum: reads require 2 replicas to respond; writes require 2 replicas to acknowledge. One replica can fail without interrupting service. ([ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md))

**Gossip membership:** each replica contacts one randomly chosen peer per second to reconcile membership histories. Two seed node addresses (stable, pre-configured) prevent the cluster from partitioning on restart.

**Client-driven routing:** for latency-sensitive paths (audit challenge dispatch, chunk assignment decisions), the service client caches cluster membership and routes directly to the responsible replica, bypassing the load balancer. This reduces 99.9th-percentile latency by 30+ ms compared to load-balancer indirection.

**Background task throttling:** background work (view refresh, repair queuing, Merkle log compaction) monitors the 99th-percentile of foreground DB read latency over the last 60 seconds. If foreground latency approaches 50 ms, background task allocation is reduced. This keeps audit challenge SLAs intact under maintenance load.

### Key API endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/provider/register` | POST | Provider registration; initiates Razorpay Linked Account creation |
| `/api/v1/provider/heartbeat` | POST | 4-hourly signed address update |
| `/api/v1/upload/assign` | POST | Returns 56 provider assignments for a new file segment |
| `/api/v1/audit/challenge` | POST | Dispatch audit challenge to a provider |
| `/api/v1/audit/receipt` | POST | Record audit receipt returned by a provider |
| `/api/v1/admin/readiness` | GET | Returns network readiness gate status |
| `/api/v1/admin/repair/queue` | GET | Repair queue depth and status |

### The cluster audit secret

Every challenge nonce is `HMAC-SHA256(server_secret, chunk_id + server_ts)`. The `server_secret` must be identical across all three replicas so that any replica can validate any challenge, including during a failover.

The secret is derived from a cluster master seed stored in the secrets manager:

```
server_secret_v{N} = HKDF-SHA256(
  ikm  = cluster_master_seed,
  salt = cluster_id,
  info = "vyomanaut-audit-secret-v" + N,
  len  = 32
)
```

Nonces are prefixed with a 1-byte version so the correct secret version is always identifiable. On rotation, the previous version is accepted for 24 hours (one full audit cycle) before being retired. ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md))

---

## 19. Reliability Scoring

Each provider has a reliability score computed from three rolling audit windows: last 24 hours (highest weight), last 7 days, and last 30 days. The score is a weighted average audit pass rate across the three windows. ([ADR-008](../decisions/ADR-008-reliability-scoring.md))

The score drives two outcomes:

**Assignment priority.** The assignment service uses Power of Two Choices: two provider candidates are drawn randomly from the vetted pool, and the higher-scored one receives the assignment. No provider is guaranteed all assignments; any vetted provider has non-zero selection probability. This prevents one provider from accumulating all new chunks (Matthew effect).

**Payment release.** The 30-day score determines the monthly release multiplier. The 7-day score triggers the deterioration signal if it drops more than 0.20 below the 30-day score.

**Scoring must be centralised.** If providers rated each other instead of being rated by the microservice, the game-theoretically dominant strategy is universal dishonesty — every provider rates every other as passing regardless of actual behaviour. The Shelby proof (Research Paper 37) formally establishes this. The microservice as sole auditor is not a convenience; it is the only architecture that produces honest outcomes.

**Bootstrapping.** New providers start with an optimistic neutral score (equivalent to BitTorrent's optimistic unchoking) and accumulate audit history through the vetting period.

---

## 20. Adversarial Defences

Five classes of adversarial provider behaviour are explicitly defended against. ([ADR-014](../decisions/ADR-014-adversarial-defences.md))

### Correlated shard placement (Honest Geppetto)

**Attack:** A group of colluding providers accumulates months of genuine trust, then deletes all their data simultaneously.

**Defence:** At assignment time, no single ASN may hold more than 20% of any file's fragments (~11 of 56). Even if the entire correlated group vanishes simultaneously, 45 fragments survive — 29 above the reconstruction threshold of 16. This cap is both a security requirement and a durability requirement independently.

### Outsourcing

**Attack:** A provider does not store the data but retrieves it just-in-time from another provider during a challenge.

**Defence:** The audit deadline is `(chunk_size / p95_measured_upload_throughput) × 1.5`. The p95 throughput is measured empirically during vetting and updated from audit responses — a provider cannot extend their deadline by self-declaring a low speed. Fetching 256 KB from another provider over the internet takes longer than the deadline allows.

### Just-in-time retrieval

**Attack:** A provider pre-caches data only when a challenge is imminent by monitoring the challenge schedule.

**Defence:** Challenge timing is randomised. The nonce is `HMAC-SHA256(server_secret, chunk_id + server_ts)` where `server_ts` is generated by the microservice at challenge time. Providers cannot predict when the next challenge arrives.

### False audit responses

**Attack:** A provider sends a plausible-looking response hash without holding the data.

**Defence:** The correct response is `SHA-256(chunk_data || challenge_nonce)`. Without the actual 256 KB chunk data, this cannot be computed correctly. The microservice independently has the expected hash and verifies it.

### Service denial

**Attack:** A provider stores the data, passes all audits, but refuses to serve retrieval requests from data owners.

**Defence:** RS(16, 56) requires more than 40 of 56 providers to simultaneously refuse retrieval before a data owner is blocked. The 20% ASN cap limits any single correlated group to ~11 providers — categorically insufficient for an effective denial. At the individual provider level, the reliability scorer should be extended to track data owner retrieval failure reports (see Q41-1 in `open-questions.md`).

---

## 21. Consistency Model

Of approximately 20 core database operations in the system, 14 are coordination-free (any replica can execute them without talking to the others) and 6 require a single authoritative executor. ([ADR-013](../decisions/ADR-013-consistency-model.md))

**Coordination-free (14 operations):** Insert audit receipt (pass or fail), increment reliability score on pass, insert provider registration (UUIDv7), insert file record, insert chunk record, all reads, increment escrow balance on deposit, soft-delete provider (status flag), trigger repair, record repair completion, issue audit challenge, validate capability token read, record repair scheduling.

**Coordinated (6 operations) — all go through the single payment microservice:**

| Operation | Why it cannot be distributed |
|---|---|
| Decrement reliability score on audit fail | Score floor ≥ 0 cannot be maintained across independent replicas |
| Decrement escrow balance on payment release | Balance floor ≥ 0 (paise) |
| Seize escrow on departure | Same floor invariant |
| Assign chunk to provider (new file upload) | Uniqueness: no two providers assigned the same slot |
| Validate capability token | Token expiry is time-dependent and cannot be checked by a replica with a stale view |
| Physical provider row deletion | Prohibited — use soft delete only |

Any new operation added to the codebase must be evaluated against this framework before choosing an implementation path. The five-step protocol: (1) state the invariant, (2) ask the merge question, (3) check Bailis Table 2, (4) scope coordination to minimum, (5) document in PR.

---

## 22. Runtime Flows

These are the five most important end-to-end flows in the system. Each describes the sequence of steps across components.

### Flow 1 — Data owner uploads a file

```
Data owner client                Microservice                   Provider (×56)
      │                               │                               │
      │─ Argon2id(passphrase) ──────► master_secret (local)          │
      │─ AONT transform (local) ───► 56 encrypted 256KB fragments    │
      │                               │                               │
      │─ POST /upload/assign ────────►│                               │
      │                               │─ Check readiness gate         │
      │                               │─ Select 56 providers          │
      │                               │  (Power of Two Choices,       │
      │                               │   ASN cap enforced)           │
      │◄─ 56 (provider, chunk_id) ───│                               │
      │                               │                               │
      │─ QUIC connect ───────────────────────────────────────────────►│ (×56 parallel)
      │─ Upload 256KB fragment ───────────────────────────────────────►│
      │◄─ Signed upload receipt ──────────────────────────────────────│
      │                               │                               │
      │─ HKDF(master_secret, file_id) ─► pointer file encryption key │
      │─ Encrypt pointer file (AEAD_CHACHA20_POLY1305) ─ local        │
      │─ POST /file/register ────────►│                               │
      │                               │─ Store encrypted ciphertext   │
      │◄─ Confirmation ──────────────│                               │
```

### Flow 2 — Daily audit challenge

```
Microservice                                          Provider
      │                                                    │
      │─ SELECT chunks due for audit ─► audit_schedule     │
      │─ server_ts = NOW()                                  │
      │─ nonce = HMAC(server_secret_vN, chunk_id+server_ts) │
      │                                                    │
      │─ QUIC dial to last_known_multiaddrs ──────────────►│ (using heartbeat address)
      │─ Send (chunk_id, nonce) ──────────────────────────►│
      │                                                    │
      │                          provider reads vLog       │
      │                          verifies content_hash     │
      │                          computes response_hash    │
      │                          signs receipt             │
      │                                                    │
      │◄─ Signed audit receipt ───────────────────────────│ (must arrive before deadline)
      │                                                    │
      │─ Verify response_hash                              │
      │─ INSERT PENDING row → audit_receipts               │
      │─ Update row: PASS/FAIL + service_sig               │
      │─ Return countersignature ─────────────────────────►│
      │─ Update reliability score windows                  │
```

### Flow 3 — Silent departure triggers repair

```
Microservice                          Replacement provider (×N missing)
      │                                          │
      │─ Heartbeat timeout: 72h elapsed          │
      │─ SET providers.status = DEPARTED         │
      │─ DELETE chunk_assignments for provider   │
      │─ Seize escrow → repair_reserve_fund      │
      │─ Enqueue repair jobs (one per chunk)     │
      │                                          │
      │─ For each affected chunk:                │
      │  ─ QUIC dial to 16 surviving holders     │
      │  ─ Download 16 fragments                 │
      │  ─ RS decode → AONT package              │
      │  ─ RS re-encode → N missing fragments    │
      │  ─ Select N replacement providers        │
      │    (ASN cap enforced)                    │
      │  ─ Upload to replacements ──────────────►│
      │◄─ Signed upload receipts ────────────────│
      │  ─ Update chunk_assignments              │
```

### Flow 4 — Monthly payment release

```
Microservice                   Razorpay              Provider's bank
      │                            │                       │
      │─ (23rd of month)           │                       │
      │─ Compute releasable balance│                       │
      │  for each active provider  │                       │
      │─ Apply release multiplier  │                       │
      │  (from 30d score)          │                       │
      │─ Check dual-window flag    │                       │
      │  (7d vs 30d deterioration) │                       │
      │                            │                       │
      │─ PATCH /transfers/:id      │                       │
      │  on_hold_until = last      │                       │
      │  working day of month ────►│                       │
      │                            │─ Release on next      │
      │◄─ Confirmation ───────────│  business day ───────►│
      │                            │                       │
      │─ Append RELEASE event      │                       │
      │  to escrow_events          │                       │
```

### Flow 5 — Data owner retrieves a file

```
Data owner client                    Provider (×16 of 56)
      │                                     │
      │─ Decrypt pointer file               │
      │  (HKDF → pointer file key →         │
      │   AEAD_CHACHA20_POLY1305 decrypt)   │
      │─ Read provider list + chunk IDs     │
      │                                     │
      │─ QUIC dial to 16 fastest providers ─────────────────────────►│
      │─ Request chunk by chunk_id ─────────────────────────────────►│
      │◄─ 256KB fragment ────────────────────────────────────────────│ (×16)
      │                                     │
      │─ Verify each chunk: SHA-256 == chunk_id
      │─ RS decode 16 fragments → AONT package
      │─ Recover K: h = SHA-256(all codewords); K = c_{s+1} XOR h
      │─ Decrypt each word: d_i = c_i XOR cipher(K, i)
      │─ Verify canary word
      │─ Reconstruct original file
```

---

## 23. Error Handling

### Audit challenge timeout

If a provider does not respond within the per-provider RTO, the microservice records `audit_result = TIMEOUT`. The reliability score decrements. The challenge is not retried until the next audit cycle (24 hours). Three consecutive TIMEOUT results in the 24-hour window do not trigger departure — only the 72-hour absence threshold does.

### Disk corruption detected by provider

If `SHA-256(chunk_data) ≠ content_hash` at read time, the provider returns `audit_result = FAIL` with a corruption-specific error code in the receipt. The microservice queues an accelerated re-audit of all chunks on that provider (the Schroeder finding: a provider with one uncorrectable error has 30× elevated probability of further errors). If multiple chunks fail within a 7-day window, the provider is treated as in rapid decline and their chunks are re-replicated on priority.

### Microservice replica failure

The (3, 2, 2) quorum absorbs single-replica failures transparently. Reads and writes continue with the two healthy replicas. Gossip membership detects the failure within seconds. The failed replica's re-integration is driven by anti-entropy: on return, it reconciles its state with gossip peers and receives any missing data via the availability service.

### Razorpay API failure

Payout failures result in a `reversed` payout state. Razorpay automatically refunds the amount to the master account. The microservice handles the `payout.reversed` webhook by leaving the provider's escrow_events row in an ATTEMPTED state and retrying on the next monthly cycle. No escrow is lost; the idempotency key prevents double-payment on retry.

### Secrets manager unavailable at startup

A microservice replica that cannot reach the secrets manager cannot load `server_secret` and will not start. This is a fail-closed design: unverifiable challenges are worse than no challenges. Existing running replicas continue serving with their cached (5-minute TTL) secret value.

### vLog write failure

If the vLog `fsync()` fails during a chunk store operation, the write is considered failed. The provider returns an error to the upload client. The upload client retries to a different provider. The incomplete vLog entry (if any) will be detected and skipped during crash recovery by validating the content_hash of the partially-written entry.

### DHT record expiry

If the availability service fails to republish a provider's DHT record within the 24-hour expiry window (12-hour republish interval, 12-hour buffer), the provider's chunk addresses expire from the DHT. Retrieval falls back to the microservice's `chunk_assignments` table. The DHT is the fast path; the microservice is the authoritative source.

---

## 24. Observability

### Key metrics (microservice)

| Metric | What it indicates |
|---|---|
| `audit_challenges_issued_total` | Challenge throughput |
| `audit_results_total{result="PASS|FAIL|TIMEOUT"}` | Network reliability |
| `provider_score_histogram` | Distribution of reliability scores |
| `repair_queue_depth` | Repair backlog |
| `repair_jobs_completed_total` | Repair throughput |
| `escrow_events_total{type="DEPOSIT|RELEASE|SEIZURE"}` | Payment volume |
| `microservice_replica_count{state="healthy"}` | Quorum health |
| `db_read_p99_latency_ms` | Foreground latency (triggers background throttle at 50 ms) |

### Key metrics (provider daemon)

| Metric | What it indicates |
|---|---|
| `chunks_stored_total` | Stored chunk count |
| `audit_responses_sent_total` | Challenge volume |
| `audit_response_latency_ms_p99` | Response time distribution |
| `vlog_append_latency_ms_p99` | Write performance |
| `content_hash_failures_total` | Silent disk corruption events |
| `heartbeat_sent_total` | Connectivity to microservice |

### Operational alerts

| Alert | Threshold | Action |
|---|---|---|
| Repair queue depth | > 1,000 jobs | Investigate departure rate; check bandwidth budget |
| TIMEOUT rate | > 5% of challenges | Check relay infrastructure; check heartbeat address freshness |
| Content hash failures | > 0 on any provider in 7-day window | Trigger accelerated re-audit of all that provider's chunks |
| Microservice healthy replicas | < 3 | Investigate replica failure |
| Release multiplier 0.00 rate | > 10% of active providers | Investigate systemic score degradation |

---

## 25. Minimum Viable Network

The system refuses upload requests until all of the following conditions are simultaneously true. The assignment service re-evaluates every 60 seconds and exposes the state at `GET /api/v1/admin/readiness`. ([ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md))

| Condition | Threshold | Reason |
|---|---|---|
| Active vetted providers | ≥ 56 | RS(16, 56) requires exactly 56 distinct shard holders per file |
| Distinct ASNs in active pool | ≥ 5 | With fewer than 5 ASNs, one ASN necessarily holds > 20% — the cap is unenforceable |
| Distinct Indian metro regions | ≥ 3 | Geographic baseline: Delhi NCR, Mumbai, and one southern metro |
| Microservice cluster state | Full (3, 2, 2) quorum | Degraded quorum is operational but not a safe launch baseline |
| Razorpay Linked Accounts | ≥ 56 with 24h cooling complete | No provider can receive payment until cooling passes |
| Relay infrastructure | ≥ 3 relay nodes deployed | Required for symmetric-NAT providers (~30% of expected population) |
| Cluster audit secret | Loaded on all replicas | All replicas must share the secret before any challenge is issued |

Upload requests return HTTP 503 ("Network not ready") until all conditions are met.

**Scale validation at 1,000,000 providers (analytical):** The Giroire BWavg formula scales as D/N (total data / provider count). For fixed per-provider storage, BWavg stays at ~39 Kbps regardless of provider count. LossRate is similarly scale-invariant for fixed D/N. The erasure parameters are valid at any scale with consistent D/N ratios.

---

## 26. Known Limitations and V3 Scope

These are explicit design decisions, each with a documented reason and a V3 path.

**No mobile providers.** At MTTF ~30–90 days, repair bandwidth exceeds the 100 Kbps background budget. Including mobile at V2 parameters would make repair consume more bandwidth than steady-state storage. Research required before V3: iOS/Android background execution limits, MTTF under financial incentives, mobile-specific erasure parameters. ([ADR-010](../decisions/ADR-010-desktop-only-v2.md))

**No public audit verification.** In V2, a data owner must trust the microservice's countersignature on audit receipts. The V3 Transparent Merkle Log will publish a daily Merkle root over all receipts, allowing independent verification without trusting the operator. ([ADR-015](../decisions/ADR-015-audit-trail.md))

**No repair bandwidth optimisation.** At V2 scale (~hundreds of providers), the 39 Kbps/peer bandwidth is comfortably inside the budget. The adoption gate for Hitchhiker codes (25–45% bandwidth reduction, V3 candidate) is: if observed V2 bandwidth exceeds 60 Kbps/peer over the first 6 months, implement. ([ADR-026](../decisions/ADR-026-repair-bw-optimisation.md))

**No upload optimality threshold.** Storj uses a parameter `o` to cancel the slowest uploads once `o` of 56 fragments confirm, reducing P99 upload latency. This is a V3 optimisation — the microservice would need to issue cancel signals to slow providers.

**Audit scalability ceiling.** Full daily audit of 100,000 providers × 10,000 chunks each approaches the ceiling of a single optimised Postgres instance (~10,000 inserts/second). V2 launches at hundreds of providers — far below this limit. If probabilistic sampling is introduced at V3 scale, the SHELBY incentive-compatibility conditions (Theorem 1, Research Paper 37) must be re-verified at the new audit frequency before deploying.

**India-only payments.** Razorpay and UPI require Indian bank accounts. The `PaymentProvider` interface in ADR-011 is designed for international gateways (e.g., Stripe Connect) to be added as implementations without rewriting payment logic.

---

## 27. ADR Reference Index

| Component | ADRs |
|---|---|
| Coordination architecture (hybrid microservice + Kademlia DHT) | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| Proof of storage / audit challenge design | [ADR-002](../decisions/ADR-002-proof-of-storage.md) |
| Erasure coding parameters (RS s=16, r=40, r0=8, lf=256 KB) | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Repair protocol (lazy, 72h threshold, priority ordering) | [ADR-004](../decisions/ADR-004-repair-protocol.md) |
| Peer selection and vetting pipeline | [ADR-005](../decisions/ADR-005-peer-selection.md) |
| Polling interval (24h) and departure threshold (72h) | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Provider exit states (four states, seizure mechanics) | [ADR-007](../decisions/ADR-007-provider-exit-states.md) |
| Reliability scoring (three rolling windows) | [ADR-008](../decisions/ADR-008-reliability-scoring.md) |
| Background CPU budget (≤5%) | [ADR-009](../decisions/ADR-009-background-execution.md) |
| No mobile providers in V2 | [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| Fiat escrow via Razorpay/UPI | [ADR-011](../decisions/ADR-011-escrow-payments.md) |
| Payment per audit passed | [ADR-012](../decisions/ADR-012-payment-basis.md) |
| Consistency model (14 coordination-free, 6 coordinated) | [ADR-013](../decisions/ADR-013-consistency-model.md) |
| Adversarial defences (five classes) | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Audit trail (signed receipts + V3 Merkle Log) | [ADR-015](../decisions/ADR-015-audit-trail.md) |
| Payment DB schema (PN-counter CRDT, escrow_events) | [ADR-016](../decisions/ADR-016-payment-db-schema.md) |
| Audit receipt schema (12 fields, Ed25519 dual signatures) | [ADR-017](../decisions/ADR-017-audit-receipt-schema.md) |
| Hot/cold storage bands | [ADR-018](../decisions/ADR-018-hot-cold-storage-bands.md) |
| Client-side cipher (ChaCha20-256 / AES-256-CTR) | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| Key management (HKDF hierarchy, pointer file, BIP-39) | [ADR-020](../decisions/ADR-020-key-management.md) |
| P2P transfer (libp2p + QUIC v1, three-tier NAT traversal) | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Encoding pipeline (AONT-RS: transform then code) | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| Provider storage engine (WiscKey: RocksDB index + vLog) | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| Economic mechanism (deterministic escrow, graduated penalty) | [ADR-024](../decisions/ADR-024-economic-mechanism.md) |
| Microservice cluster quorum ((3,2,2) + gossip) | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) |
| Repair bandwidth optimisation (V3 — Hitchhiker candidate) | [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) |
| Cluster audit secret (HKDF derivation, versioned rotation) | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |
| Provider heartbeat (4-hour address update) | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| Bootstrap minimum viable network (7 conditions) | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |