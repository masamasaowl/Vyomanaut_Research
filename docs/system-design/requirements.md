# Vyomanaut V2 — Product Requirements Document

**Status:** Draft  
**Author:** Vyomanaut Engineering  
**Date:** April 2026  
**Version:** 1.0  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
**Architecture reference:** `docs/system-design/architecture.md`  
**Decisions index:** `docs/decisions/README.md`

---

## 1. Overview

This document formalises what Vyomanaut V2 must do and how well it must do it, in a form
that product, engineering, and operations teams can align on before a single line of
production code is committed. V1 failed because it was built without this alignment — the
architecture document describes what V2 is; this PRD describes what V2 must do for the
people who use it. Every requirement here has a traceable root in either a user story or
an accepted ADR. Where the two conflict, the ADR wins.

---

## 2. Problem Statement

**User problem — data owners:** Cloud storage (AWS S3, GCS, iCloud) is either expensive at
scale, proprietary, or geographically centralised. Data owners — individuals, small businesses,
creators — pay for storage they do not always trust, at prices set by monopolies, knowing
their unencrypted files sit on servers they do not control.

**User problem — storage providers:** Millions of Indian home desktop owners and NAS operators
have idle terabytes of disk space and symmetrical 100 Mbps connections at ₹600/month. No
product lets them monetise this idle capacity reliably, transparently, and without technical
expertise.

**Business problem:** Vyomanaut V1 confirmed both markets exist but was too unreliable (slow
peer discovery, no audit enforcement, no payment guarantees) to retain either user type past
the first week. V2 is a research-first rebuild that solves the reliability and trust problems
structurally before building any user-facing surface.

**Evidence:**
- V1 post-mortem: lack of research-backed architecture drove structural compromises within
  the first sprint. Three failure modes identified: inefficient peer discovery, no audit
  enforcement, and no payment guarantees.
- IPFS production measurement (Paper 20): 87.6% of unincentivised P2P sessions last under
  8 hours. Financial friction is necessary to sustain provider uptime.
- Razorpay API availability: UPI has zero merchant per-transaction fee and settles in
  seconds — the payment rail exists and is accessible to any Indian bank account holder.

---

## 3. Goals and Non-Goals

### Goals

- A data owner can upload, store, and retrieve files knowing no party in the system can read
  them — not Vyomanaut, not the providers, not a network observer.
- A provider can earn a predictable monthly income in Indian rupees for keeping a daemon
  running and passing storage audits, without understanding cryptography.
- The network sustains data durability above 10⁻¹⁵ annual loss probability per file even
  as providers join and leave daily.
- Every provider interaction — registration, audit response, payment, departure — has a
  defined outcome that both parties can verify independently.

### Non-Goals

- **Mobile providers in V2.** Operating system background execution limits on iOS and Android
  make provider-grade uptime impossible without special permissions that most users will not
  grant. Mobile providers are deferred to V3. (ADR-010)
- **File versioning or in-place updates.** Files are immutable once stored. Updating a file
  means deleting and re-uploading. Version history is out of scope.
- **Content deduplication across data owners.** Each upload uses a fresh AONT key. Two data
  owners storing the same file consume 2× the network capacity. This is the privacy price.
- **A retrieval marketplace.** Providers are not paid for serving downloads. Payment is for
  sustained storage presence proved by daily audits, not for bandwidth consumed.
- **International payments at launch.** V2 is India-only. Razorpay and UPI require Indian
  bank accounts. International payment support is deferred to a later version. (ADR-011)
- **A web dashboard for providers.** V2 ships a desktop daemon and a CLI. Provider UI is
  deferred to V3.

---

## 4. User Personas

### Persona A — The Data Owner ("Meera")

Meera runs a small independent film production studio in Pune. She generates 50–200 GB of
raw footage per project and needs affordable, long-term archive storage. She is technically
literate enough to install desktop software and follow a setup wizard, but not enough to
manage encryption keys manually. She cares deeply about privacy — she has been burned by a
cloud provider's breach before — and about cost.

**Jobs to be done:**
- Store large files cheaply and permanently without trusting a centralised operator.
- Retrieve any file on demand within minutes.
- Know that if her laptop dies, she can recover her files with a passphrase.

### Persona B — The Storage Provider ("Ravi")

Ravi works in IT at a mid-sized company in Bangalore. He has a home NAS with 4 TB free and
a 100 Mbps symmetrical BSNL connection. He found Vyomanaut through a tech forum and wants to
earn ₹500–2,000 per month passively. He expects it to "just work" in the background and
deposit money at the start of each month.

**Jobs to be done:**
- Install the daemon once and forget about it.
- See his earnings grow predictably without manual intervention.
- Trust that his earnings are protected if he needs to take his machine offline for a week.

---

## 5. User Stories

### Data Owner Stories

**DO-01** — As a data owner, I want to upload a file from my desktop and receive confirmation
that it has been distributed across providers, so that I know my data is protected before I
close the app.

**DO-02** — As a data owner, I want to retrieve any previously uploaded file by entering my
passphrase, so that I am not dependent on any single provider or device to access my data.

**DO-03** — As a data owner, I want to delete a file and receive confirmation that all 56
fragments have been removed from the network, so that I stop being billed for it.

**DO-04** — As a data owner, I want to see my storage usage, monthly cost, and active file
list in a single view, so that I can manage my spend without opening a spreadsheet.

**DO-05** — As a data owner, I want to recover all my files on a new machine by entering
my passphrase (or a 24-word recovery phrase), so that a stolen or broken laptop is not a
data loss event.

**DO-06** — As a data owner, I want to deposit funds into my escrow account via UPI, so
that I can pay for storage without entering credit card details or creating a crypto wallet.

**DO-07** — As a data owner, I want the upload to use only my local device for encryption,
so that I can verify that the service never sees my plaintext data.

### Provider Stories

**PR-01** — As a storage provider, I want to install the daemon with a single installer
and have it register automatically, so that I do not need to configure networking or
cryptography.

**PR-02** — As a storage provider, I want to see my current earnings, upcoming release date,
and reliability score on a dashboard, so that I know my daemon is working without tailing
log files.

**PR-03** — As a storage provider, I want to announce a planned offline period and have the
system hold my earnings rather than penalise me, so that I can take my machine for repairs
without financial consequence.

**PR-04** — As a storage provider, I want to receive my monthly earnings automatically to
my UPI-linked bank account, so that I do not need to initiate a withdrawal.

**PR-05** — As a storage provider, I want to set a maximum storage allocation (e.g., 500 GB)
so that the daemon never fills my disk beyond what I am comfortable offering.

**PR-06** — As a storage provider joining the network, I want to understand what my expected
monthly earnings are before committing disk space, so that I can decide if participation is
worth it.

**PR-07** — As a storage provider, I want to announce my departure from the network and have
the system gracefully migrate my chunks before I shut down, so that I receive my remaining
earnings rather than a seizure.

### System / Operator Stories

**SY-01** — As an operator, I want the network to refuse data owner uploads until the
minimum viable provider pool is satisfied, so that no file is stored with insufficient
redundancy from day one. (ADR-029)

**SY-02** — As an operator, I want every audit receipt to be signed by both the provider
and the microservice, so that I can produce a receipt in any dispute without relying on
either party's self-report. (ADR-015, ADR-017)

**SY-03** — As an operator, I want the system to automatically trigger repair within 72
hours of a provider's silent departure, so that no file sits below the repair threshold
waiting for manual intervention. (ADR-004, ADR-007)

---

## 6. Functional Requirements

Requirements are grouped by domain. Each carries a priority label: **P0** (must ship for
V2 launch), **P1** (must ship before first paying customer), **P2** (should ship within the
first quarter post-launch). Every P0 requirement is a launch blocker.

### 6.1 Data Owner — Registration and Onboarding

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-001 | The system must allow a data owner to create an account using a phone number verified by OTP, with no other personal information required. | P0 | Registration gate is the Sybil defence |
| FR-002 | At account creation, the system must derive a 32-byte master secret from the owner's passphrase using Argon2id (t=3, m=64 MB, p=4) and must never write the master secret to disk or transmit it to the microservice. | P0 | ADR-020 |
| FR-003 | At account creation, the system must generate a BIP-39 24-word mnemonic from the master secret and display it exactly once, requiring the owner to confirm two randomly chosen words before proceeding. | P0 | ADR-020 — only recovery path on passphrase loss |
| FR-004 | The system must allow a data owner to restore access on a new device using either their passphrase or their 24-word BIP-39 mnemonic. | P0 | ADR-020 |
| FR-005 | The system must warn the data owner, in plain language, that loss of both the passphrase and the mnemonic results in permanent, unrecoverable data loss with no support path. | P0 | UX requirement; legal protection |
| FR-006 | The system must allow a data owner to deposit escrow funds using UPI Intent flow (not UPI Collect, which is deprecated as of February 2026), with Smart Collect 2.0 reconciliation. | P0 | ADR-011, Paper 35 |

### 6.2 Data Owner — File Upload

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-007 | The client must perform all encryption and erasure encoding on the data owner's device before any data is transmitted to any provider. | P0 | ADR-019, ADR-022 |
| FR-008 | The encoding pipeline must use AONT-RS: apply the All-or-Nothing Transform (ChaCha20-256 on hardware without AES-NI, AES-256-CTR with AES-NI) followed by systematic Reed-Solomon coding with s=16, r=40, producing 56 fragments of 256 KB each. | P0 | ADR-022, ADR-019, ADR-003 |
| FR-009 | The system must select 56 distinct providers for each file segment, ensuring no single ASN holds more than 20% of fragments (~11 of 56) for that segment, and must refuse the upload if this constraint cannot be satisfied. | P0 | ADR-014, ADR-005 |
| FR-010 | The system must upload all 56 fragments via direct libp2p/QUIC P2P connections to providers, without routing any file data through the microservice. | P0 | ADR-021; data plane / control plane separation |
| FR-011 | The system must create a pointer file containing the 56 provider IDs, 56 chunk content addresses, and erasure parameters, encrypt it with AEAD_CHACHA20_POLY1305 (key derived via HKDF from the master secret), and store the encrypted ciphertext with the microservice. | P0 | ADR-020, ADR-022 |
| FR-012 | The system must display upload progress per fragment (not just total), allowing the data owner to see when the upload is complete even if individual providers respond slowly. | P1 | UX requirement |
| FR-013 | The system must display the expected monthly storage cost for the file before the upload begins, calculated from the current storage rate and file size. | P1 | DO-04, FR-006 alignment |
| FR-014 | The system must refuse to begin an upload if the data owner's escrow balance is insufficient to cover 30 days of storage for the file. | P0 | Prevents free-riding on provider capacity |

### 6.3 Data Owner — File Retrieval

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-015 | The client must retrieve the encrypted pointer file from the microservice, decrypt it locally using the HKDF-derived key, and use the provider list and chunk IDs inside to initiate retrieval — without the microservice being involved in the data transfer. | P0 | ADR-020, zero-knowledge |
| FR-016 | The client must attempt retrieval from the 16 fastest-responding providers (not necessarily the first 16 in the pointer file), cancelling remaining connections once k=16 responses are received. | P1 | Reduces P99 retrieval latency |
| FR-017 | The client must verify each retrieved fragment against its SHA-256 content address before decoding. Any fragment that fails must be replaced by requesting from an alternate provider. | P0 | Integrity check; ADR-022 canary verification |
| FR-018 | After RS decoding and AONT decryption, the client must verify the canary word. If it fails, the client must surface an error and must not hand corrupted plaintext to the data owner. | P0 | ADR-022 |

### 6.4 Data Owner — File Management

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-019 | The system must provide a file list view showing each file's name, size, upload date, current storage cost per month, and retrieval status (all fragments available / degraded / unavailable). | P1 | DO-04 |
| FR-020 | The system must allow a data owner to delete a file, which must trigger removal of all 56 chunk assignments from the microservice and notify each holding provider to delete their fragment. | P1 | ADR-007; GC on vLog |
| FR-021 | The system must provide an escrow balance view showing: current balance, amount reserved for active files (next 30 days), amount available for withdrawal, and transaction history. | P1 | DO-04, ADR-016 |

### 6.5 Provider — Installation and Registration

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-022 | The provider daemon must be installable on Windows 10+, macOS 12+, and Ubuntu 22.04+ via a signed single-file installer with no manual dependency installation. | P0 | ADR-009; desktop-only V2 |
| FR-023 | At first launch, the daemon must generate an Ed25519 key pair, persist it encrypted under a daemon-local passphrase, and display the public key fingerprint to the provider for their records. | P0 | ADR-021; peer identity |
| FR-024 | The daemon must register with the microservice by submitting the provider's phone number (OTP-verified), Ed25519 public key, declared storage allocation in GB, and declared city/region. | P0 | ADR-001, ADR-005 |
| FR-025 | The system must initiate Razorpay Route Linked Account creation asynchronously after registration and must not begin chunk assignment until the 24-hour cooling period has elapsed and the account is confirmed active. | P0 | ADR-024, Paper 35 |
| FR-026 | The daemon must set `providers.status = PENDING_ONBOARDING` at registration, advance to `VETTING` on first successful heartbeat, and advance to `ACTIVE` automatically once 80 consecutive audit passes are recorded. | P0 | ADR-005, ADR-007 |

### 6.6 Provider — Operation

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-027 | The daemon must send a signed heartbeat to the microservice control plane every 4 hours containing the provider's current libp2p multiaddresses, so that the microservice always has a fresh address for challenge dispatch. | P0 | ADR-028; DHCP rotation problem |
| FR-028 | The daemon must respond to audit challenges by reading the specified chunk from the vLog, verifying its SHA-256 content hash, computing SHA-256(chunk_data ‖ challenge_nonce), and returning a signed receipt — all within the per-provider deadline of (256 KB / p95_throughput_kbps) × 1.5. | P0 | ADR-002, ADR-014 |
| FR-029 | The daemon must expose a local status interface (CLI or tray icon) showing: daemon health, number of stored chunks, last audit result, current reliability score, and pending earnings. | P1 | PR-02 |
| FR-030 | The daemon must honour a provider-set storage cap: if accepting a new chunk assignment would exceed the cap, the daemon must decline the assignment and inform the microservice. | P1 | PR-05 |
| FR-031 | The daemon must auto-start on OS boot using the platform-appropriate mechanism (Windows Service, macOS LaunchDaemon, Linux systemd unit) with no manual configuration by the provider. | P0 | ADR-009; reliability of provider uptime |

### 6.7 Provider — Exit and Departure

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-032 | The system must allow a provider to announce a planned offline period (0–72 hours) through the app. During a promised offline period, audit FAILs must not decrement the reliability score, and no escrow penalty must be applied unless the period is overrun. | P0 | ADR-007 |
| FR-033 | If a provider overruns a promised offline period, the system must reclassify the state to Permanent Silent Departure and apply the 72-hour departure consequences. | P0 | ADR-007 |
| FR-034 | The system must allow a provider to announce a permanent departure. On announcement, the system must immediately queue repair for all affected chunks, release the provider's pending escrow proportional to the fraction of the current 30-day window completed, and set `providers.status = DEPARTED`. | P0 | ADR-007, ADR-024 |
| FR-035 | When a provider's last heartbeat exceeds 72 hours, the system must automatically declare a silent departure, seize the 30-day rolling escrow window into the repair reserve fund, trigger repair for all chunks, and block the provider's Peer ID from further interactions. | P0 | ADR-007, ADR-024 |
| FR-036 | A provider declared as silently departed who attempts to reconnect must receive HTTP 403. Re-joining the network requires a new full registration with a new phone number. | P0 | ADR-007; prevents gaming the departure threshold |

### 6.8 Audit System

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-037 | The microservice must issue exactly one audit challenge per assigned chunk per 24-hour period, with challenge timing randomised within the window to prevent providers from anticipating it. | P0 | ADR-002, ADR-014 |
| FR-038 | Every challenge nonce must be generated as HMAC-SHA256(server_secret_vN, chunk_id + server_ts) where server_ts is set by the microservice at challenge time. Providers must not be able to compute nonces in advance. | P0 | ADR-017, ADR-027 |
| FR-039 | Every audit event must produce an INSERT-only row in `audit_receipts` containing the 12 schema fields defined in ADR-017, signed by both provider (Ed25519) and microservice (Ed25519). No row in this table may ever be updated or deleted. | P0 | ADR-015, ADR-017 |
| FR-040 | The microservice must use a per-provider TCP-style RTO (RTO = AVG + 4 × VAR of recent audit response times) as the challenge timeout, not a fixed value. New providers must default to the pool-median RTO. | P0 | ADR-006, Paper 28 |
| FR-041 | If a provider's content_hash verification fails when reading a chunk for an audit response, the provider daemon must immediately report `audit_result = FAIL` with a corruption error code, and the microservice must queue accelerated re-audit of all chunks on that provider within the next polling cycle. | P0 | ADR-023, Paper 32 |

### 6.9 Repair System

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-042 | The repair scheduler must trigger a repair job when the available fragment count for any chunk drops to s + r0 = 24 (8 above the reconstruction floor of s=16). | P0 | ADR-004 |
| FR-043 | The repair scheduler must prioritise repair jobs triggered by confirmed permanent departures (72-hour threshold) over jobs triggered by the r0 pre-warning threshold. Transient-absence jobs must wait behind permanent-departure jobs. | P0 | ADR-004, Paper 39 |
| FR-044 | Repair must be triggered immediately (regardless of the 72-hour threshold) if the fragment count for any chunk drops to s=16 (the reconstruction floor). | P0 | ADR-004 emergency floor |
| FR-045 | During repair, replacement providers must be selected with the same 20% ASN cap constraint that governs original chunk assignment. | P0 | ADR-014 |

### 6.10 Payment System

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-046 | All payment amounts must be computed and stored as integer paise (₹1 = 100 paise). Floating-point arithmetic must not appear anywhere in the payment calculation or storage path. | P0 | ADR-016 |
| FR-047 | Every payout API call to Razorpay must include the `X-Payout-Idempotency` header set to SHA-256(provider_id + audit_period). Duplicate payout calls with the same key must return the existing payout state without creating a second transfer. | P0 | ADR-012, Paper 35 |
| FR-048 | Monthly earnings release must run on the 23rd of each month. Razorpay's `on_hold_until` must be set to the last working day of the current month (accounting for RBI bank holidays via a static lookup table updated each December), targeting release within the first 3 business days of the following month. | P0 | ADR-024, Paper 35 |
| FR-049 | The release multiplier must be applied per provider based on their 30-day reliability score: 1.00 (score ≥ 0.95), 0.75 (0.80–0.94), 0.50 (0.65–0.79), 0.00 (< 0.65). Withheld portions must roll into the next month's escrow window, not be seized. | P0 | ADR-024 |
| FR-050 | If the 7-day reliability score drops more than 0.20 below the 30-day score, the dual-window flag must be set and the next release must use the lower multiplier of the two score windows. | P0 | ADR-024, Paper 31 |
| FR-051 | During the vetting period (first 4–6 months), the hold window must be 60 days and the release cap must be 50%. After vetting is complete, the hold window must revert to 30 days with no release cap. | P0 | ADR-024 |
| FR-052 | The microservice must refuse any request to transfer a provider's escrow balance to a different provider_id. Escrow is identity-bound and non-transferable. | P0 | ADR-024, Paper 33 |

### 6.11 Network Readiness Gate

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-053 | The assignment service must return HTTP 503 for all data owner upload requests until all seven readiness conditions are simultaneously satisfied: ≥ 56 active vetted providers, ≥ 5 distinct ASNs, ≥ 3 distinct Indian metro regions, full (3,2,2) microservice quorum, ≥ 56 Razorpay Linked Accounts with 24-hour cooling complete, ≥ 3 relay nodes deployed, and the cluster audit secret loaded on all replicas. | P0 | ADR-029 |
| FR-054 | The microservice must expose a `GET /api/v1/admin/readiness` endpoint that returns the current pass/fail status of each of the seven readiness conditions, re-evaluated every 60 seconds. | P0 | ADR-029; operational monitoring |

### 6.12 Provider Daemon — Simulation Mode

| ID | Requirement | Priority | ADR / Notes |
|----|-------------|----------|-------------|
| FR-055 | The provider daemon must support a `--sim-count=N` flag that launches N simulated provider instances in a single process, each with isolated key pairs, RocksDB instances, and vLog files, for local integration testing without physical machines. | P0 | ADR-029; cannot build without testability |
| FR-056 | Simulation mode must not bypass the network readiness gate; a simulation with `--sim-count=56` and `--sim-asn-count=5` must be required to satisfy the same readiness conditions as production before uploads are permitted. | P0 | ADR-029; simulation must proxy production behaviour |

---

## 7. Non-Functional Requirements

### 7.1 Durability

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-001 | The system must sustain an annual file loss rate of less than 10⁻¹⁵ per stored file at the target provider MTTF of 300 days. | Durability | < 10⁻¹⁵/year | ADR-003 |
| NFR-002 | The 20% ASN cap (FR-009) is a co-requisite for NFR-001. Disabling the cap invalidates the durability guarantee regardless of erasure parameters. | Durability | Non-negotiable | ADR-003, ADR-014 |
| NFR-003 | The system must tolerate the simultaneous departure of any 40 of 56 fragment holders for any file without data loss or reconstruction failure. | Durability | 40-fault tolerance | ADR-003 |

### 7.2 Availability

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-004 | A file must be retrievable as long as any 16 of its 56 fragment holders are reachable. The client must retry across alternate providers without user intervention. | Availability | k=16 of n=56 | ADR-003, FR-016 |
| NFR-005 | The coordination microservice must absorb the failure of any single replica without interrupting audit scheduling, challenge dispatch, or payment processing. | Availability | Single-replica fault tolerance | ADR-025 |
| NFR-006 | A provider behind symmetric NAT (estimated 30% of Indian residential ISPs) must remain fully auditable via Circuit Relay v2 fallback, with relay overhead below 50 ms from Indian cloud-hosted relay nodes. | Availability | Relay RTT < 50 ms | ADR-021, Paper 30 |

### 7.3 Performance

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-007 | Each provider must respond to an audit challenge within (256 KB / p95_measured_upload_throughput_kbps) × 1.5. For a provider with p95 throughput of 500 KB/s this is 768 ms. | Performance | Per-provider deadline | ADR-014 |
| NFR-008 | The audit challenge lookup path on the provider daemon (Bloom filter check + RocksDB lookup + vLog read + hash verification) must complete within 100 ms at p99 on SSD hardware and 200 ms at p99 on HDD hardware, under concurrent upload load. | Performance | p99 ≤ 100 ms SSD / 200 ms HDD | ADR-023 |
| NFR-009 | The AONT encoding pass for a full 14 MB segment must complete within 200 ms at p50 and 400 ms at p99 on hardware without AES-NI (minimum-spec Indian desktop: dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD). | Performance | p50 ≤ 200 ms | ADR-019, benchmarking-protocol.md |
| NFR-010 | The Argon2id master secret derivation at session start (t=3, m=64 MB, p=4) must complete within 500 ms at p50 on the minimum-spec target hardware. If it does not, parameters must be reduced per the fallback protocol in benchmarking-protocol.md. | Performance | p50 ≤ 500 ms | ADR-020 |
| NFR-011 | The provider daemon must consume no more than 5% of CPU and remain within normal desktop I/O load during steady-state audit and transfer operation. | Performance | ≤ 5% CPU | ADR-009 |
| NFR-012 | Steady-state repair bandwidth per provider must not exceed 100 Kbps at the target MTTF of 300 days and a network of 1,000 providers each storing 50 GB. At these parameters, BWavg ≈ 39 Kbps/peer per the Giroire formula. | Performance | ≤ 100 Kbps/provider | ADR-003, ADR-004 |
| NFR-013 | Write amplification in the provider storage engine must not exceed 1.1× at 256 KB chunk size, meaning a provider storing 50 GB of chunks must write no more than 55 GB to their storage device in total. | Performance | Write amplification ≤ 1.1× | ADR-023 |

### 7.4 Security and Privacy

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-014 | The service must never hold, derive, or transmit any key that could decrypt any stored file. The microservice stores pointer file ciphertext but must never hold the decryption key. | Security | Zero-knowledge | ADR-019, ADR-020 |
| NFR-015 | All audit challenge responses must be unforgeable without the actual chunk data. The correct response (SHA-256(chunk_data ‖ nonce)) cannot be computed by a provider who has deleted the chunk. | Security | Computational unforgability | ADR-002 |
| NFR-016 | All provider-to-provider and provider-to-client connections must be authenticated at the transport layer (TLS 1.3 via QUIC, or Noise XX via TCP). A provider cannot impersonate another provider's Peer ID. | Security | Transport authentication | ADR-021 |
| NFR-017 | DHT lookup keys must be pseudonymous: chunk lookup uses HMAC-SHA256(chunk_hash, file_owner_key) so that a DHT observer cannot correlate lookup traffic with file identity. | Security / Privacy | DHT privacy | ADR-001 |
| NFR-018 | The cluster audit secret (server_secret) must be derived via HKDF from a cluster master seed stored in a secrets manager (Vault, AWS SSM, or GCP Secret Manager). It must never be written to disk in plaintext and must never be transmitted between replicas in cleartext. | Security | Key management | ADR-027 |
| NFR-019 | Poly1305 tag comparison in the pointer file decryption path must use constant-time comparison. A timing oracle on tag verification must not be introduced. | Security | Timing attack prevention | ADR-019 |
| NFR-020 | All challenge nonces must include a 1-byte version prefix identifying which server_secret version was used, so that any replica can validate any nonce across failover without coordination. | Security | Cross-replica correctness | ADR-027 |

### 7.5 Reliability and Correctness

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-021 | The audit receipt table must be INSERT-only. No UPDATE or DELETE must be possible from any application path. This must be enforced at the Postgres row security policy level, not only by application code. | Correctness | Tamper-evident log | ADR-015 |
| NFR-022 | The escrow_events table must be INSERT-only. Account balance must always be computed from the event log (SUM of deposits minus releases and seizures) and must never be stored as a mutable column. | Correctness | CRDT-safe ledger | ADR-016 |
| NFR-023 | The vLog write path on the provider daemon must serialise all appends through a single writer goroutine. Concurrent upload goroutines must not write to the vLog file handle directly. | Correctness | Data integrity | ADR-023 |
| NFR-024 | On provider daemon crash and restart, the daemon must scan the vLog tail and re-insert any index entries missing from RocksDB before accepting new audit challenges or upload requests. | Correctness | Crash recovery | ADR-023 |

### 7.6 Observability and Operability

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-025 | The microservice must expose the following metrics (at minimum) to a Prometheus-compatible scrape endpoint: audit challenges issued, audit results by outcome (PASS/FAIL/TIMEOUT), repair queue depth, repair jobs completed, escrow events by type, microservice replica count by health state, and foreground DB read p99 latency. | Observability | Prometheus | ADR-025 |
| NFR-026 | The provider daemon must expose the following metrics to the local status interface: stored chunk count, audit response p99 latency, content hash failure count, heartbeat success rate, and pending earnings in paise. | Observability | Local daemon | ADR-009 |
| NFR-027 | The system must fire an alert if any of the following thresholds are crossed: repair queue depth > 1,000 jobs, audit TIMEOUT rate > 5% of challenges in a 1-hour window, content hash failure count > 0 on any provider in a rolling 7-day window, microservice healthy replica count < 3. | Operability | Alert thresholds | architecture.md §24 |
| NFR-028 | Background tasks in the microservice (view refresh, repair queuing, Merkle log compaction) must monitor the 99th-percentile of foreground DB read latency over the last 60 seconds. If it approaches 50 ms, background task allocation must be reduced. | Operability | Background throttling | ADR-025 |

### 7.7 Compliance and Payments

| ID | Requirement | Type | Target | ADR |
|----|-------------|------|--------|-----|
| NFR-029 | All UPI deposit flows must use UPI Intent (app-based selection) or QR code. UPI Collect flow must not be used. (UPI Collect is deprecated by NPCI as of 28 February 2026.) | Compliance | NPCI mandate | ADR-011, Paper 35 |
| NFR-030 | Every Razorpay payout API call must include the `X-Payout-Idempotency` header (mandatory as of 15 March 2025). Payout calls without this header are rejected by Razorpay. | Compliance | Razorpay API | ADR-012, Paper 35 |
| NFR-031 | The RBI bank holiday lookup table used to compute `on_hold_until` dates must be updated as part of the December release deployment each year. | Compliance | RBI | ADR-024 |

---

## 8. UX Considerations

### 8.1 Critical UX Moments

**Mnemonic backup (FR-003):** This is the highest-stakes UX moment in the product. If a
data owner does not back up their mnemonic and later loses their passphrase, they lose their
data permanently. The UI must:
- Present the 24 words on a single screen with no surrounding UI noise.
- Explicitly state "These words are the ONLY way to recover your files if you forget your
  passphrase. Write them down now."
- Block progression until the owner correctly enters at least two randomly selected words.
- Never offer to store the mnemonic in the app, email it, or copy it to the clipboard
  automatically.

**Storage cost transparency (FR-013):** Providers see a storage rate in paise per GB per
month. Data owners see a monthly cost in rupees. Both must be derived from the same
underlying rate with no hidden fees. The cost display must update in real time as the data
owner selects files.

**Provider earnings dashboard (FR-029):** The provider's primary question every day is "is
my daemon working and am I earning?" The UI must answer this on a single screen without
the provider needing to understand reliability scores, escrow windows, or audit mechanics.
Translate technical state to plain language:
- "Your daemon is healthy. You've earned ₹340 this month."
- "Your machine was offline for 26 hours. Your score dropped slightly but your earnings
  are not affected."
- "Warning: your machine has been offline for 60 hours. Earnings may be held if it does
  not reconnect within 12 hours."

### 8.2 Edge States

**Empty state — new data owner with no files:** Show estimated cost for a hypothetical 100 GB
upload, a link to add escrow funds, and an upload button. Do not show a blank screen.

**Degraded file state (some fragments unavailable):** Show which file is degraded, explain
in plain language that repair is in progress, and give an estimated completion time. Do not
surface the fragment count or erasure parameters.

**Escrow balance too low:** When a data owner's escrow balance will run out within 7 days,
show a non-blocking banner with the top-up amount and a UPI deep link. When it runs out,
block new uploads (not retrieval) and show a clear error.

**Provider daemon not running:** If the daemon has not sent a heartbeat in 4 hours, the
provider dashboard must detect this via the microservice's last_heartbeat_ts and display
a prominent warning with instructions to restart the daemon.

---

## 9. Technical Considerations

### 9.1 Hard Constraints

- The encoding pipeline runs entirely on the data owner's device. Any cloud-offloaded
  encoding path (e.g., Szabó et al. proxy model, Paper 15) is explicitly rejected for V2
  because Vyomanaut has no ISP operator relationships. (Paper 15 break)
- Razorpay Escrow+ is not available to Vyomanaut — it requires NBFC registration. Route
  with `on_hold` is the only available partial-hold primitive. (Paper 35)
- All amounts must be integer paise. Float arithmetic in the payment path is a correctness
  violation, not just a style concern. (ADR-016)

### 9.2 Key External Dependencies

| Dependency | Failure mode | Impact |
|------------|-------------|--------|
| Razorpay Route API | Payment releases pause | Audits and storage continue; providers experience delayed payments |
| Secrets manager (Vault / SSM) | Replicas cannot start | Challenge issuance halts on startup; existing instances run with cached secret for 5 minutes |
| Indian ISP infrastructure | Connectivity degradation | Covered by 40-fragment parity and relay infrastructure |
| RBI bank holiday calendar | Wrong release dates | Static table updated annually in December deployment |

### 9.3 Data Model Implications

- `audit_receipts`: INSERT-only, row security policy enforced at DB. Unique index on
  `challenge_nonce` for idempotent retry. `audit_result` column must accept NULL (in-flight
  state). `abandoned_at` column for GC of stale PENDING rows. Schema version must be 33
  bytes for `challenge_nonce` (32-byte HMAC + 1-byte version prefix per ADR-027).
- `providers`: `last_known_multiaddrs JSONB`, `last_heartbeat_ts TIMESTAMPTZ`,
  `multiaddr_stale BOOLEAN` added per ADR-028.
- `escrow_events`: INSERT-only, idempotency_key UNIQUE, amount_paise BIGINT only.

### 9.4 Benchmark Requirements Before Shipping

The following benchmarks from `docs/research/benchmarking-protocol.md` must pass on a
minimum-spec machine before V2 launches:

- **Q16-1:** AONT encoding throughput — p50 ≤ 200 ms per 14 MB segment without AES-NI.
- **Q18-1:** Argon2id session start latency — p50 ≤ 500 ms at t=3, m=64 MB.
- **Q27-1:** RocksDB rate limiter calibration — p99 audit latency ≤ 100 ms (SSD) at the
  highest compaction rate that does not violate it.
- **HDD-specific audit latency** (ADR-023): p99 ≤ 200 ms under active compaction on a
  7200 RPM consumer HDD.

---

## 10. Analytics and Instrumentation

### 10.1 Primary Metric

**Data owner 30-day retention** — the fraction of data owners who have at least one active
file stored 30 days after their first upload. Target: ≥ 70% in the first 90 days of launch.

### 10.2 Provider Primary Metric

**Provider 90-day survival rate** — the fraction of registered providers who are still ACTIVE
(not DEPARTED) 90 days after their first chunk assignment. Target: ≥ 60%. This is the proxy
for MTTF validation. (Q08-1)

### 10.3 Key Events

| Event | Trigger | Purpose |
|-------|---------|---------|
| `data_owner_registered` | OTP verified, account created | Funnel top |
| `mnemonic_confirmed` | Owner correctly enters 2 of 24 words | Safety gate completion rate |
| `escrow_deposit_initiated` | UPI Intent flow started | Payment funnel |
| `escrow_deposit_confirmed` | Razorpay webhook received | Revenue recognition |
| `file_upload_started` | Encoding pipeline begins | Upload funnel top |
| `file_upload_completed` | All 56 upload receipts received | Upload conversion |
| `file_upload_failed` | Any hard error (network, provider, escrow) | Error analysis |
| `file_retrieved` | Canary verified, plaintext delivered | Retrieval success |
| `provider_registered` | OTP verified, Ed25519 key submitted | Provider funnel top |
| `provider_vetting_completed` | 80th consecutive audit pass | Vetting conversion |
| `provider_departed_announced` | Announced departure received | Voluntary churn |
| `provider_departed_silent` | 72h threshold crossed | Involuntary churn |
| `audit_pass` | PASS receipt countersigned | Core reliability signal |
| `audit_fail` | FAIL receipt countersigned | Reliability degradation |
| `audit_timeout` | RTO exceeded | Address / connectivity issue |
| `repair_job_queued` | Fragment count at r0 threshold | Durability health |
| `repair_job_completed` | Replacement fragments uploaded | Durability health |
| `payment_released` | Payout processed | Revenue distributed |
| `payment_seized` | Silent departure escrow seized | Financial SLA enforcement |

### 10.4 Guardrail Metrics (must not worsen)

- **Content hash failure rate per provider per day** — must remain < 1% across the fleet.
  Spikes indicate a cohort of failing disks.
- **Relay-dependent provider fraction** — must not exceed 45%. If it does, add relay
  infrastructure before the constraint becomes a reliability risk. (ADR-021, Q20-1)
- **Repair queue depth** — must not exceed 5,000 jobs. Above this, repair bandwidth is
  likely to exceed the 100 Kbps/provider budget.

---

## 11. Launch Plan

### 11.1 Phases

| Phase | Condition to exit | Upload gate |
|-------|-----------------|-------------|
| **Internal** | All P0 FRs and NFR benchmarks pass | Disabled (internal team only) |
| **Private beta** | Network readiness gate satisfied (FR-053); relay nodes deployed | Enabled for 20 invited data owners and 100 providers |
| **Public beta** | Provider 30-day survival rate ≥ 50%; no data loss events; audit TIMEOUT rate < 5% | Open registration, escrow deposits enabled |
| **V2 GA** | Provider 90-day survival rate ≥ 60%; data owner 30-day retention ≥ 70% | Full public |

### 11.2 Feature Flags

| Flag | Default (internal) | Default (beta) | Purpose |
|------|--------------------|---------------|---------|
| `upload_gate_enabled` | false | true | Enforces FR-053 readiness conditions |
| `payment_releases_enabled` | false | true | Enables actual Razorpay payouts |
| `sim_mode_allowed` | true | false | Allows `--sim-count` flag on daemon |
| `provider_ui_enabled` | false | false | Enables provider dashboard (P1 feature) |

### 11.3 Risk Factors and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Indian ISP CGNAT blocks QUIC for > 30% of providers | Medium | High | Confirmed TCP fallback path; identical hole-punch success rate (Paper 30) |
| Razorpay Route API changes break `on_hold_until` semantics | Low | High | Abstract behind PaymentProvider interface (ADR-011); monitor API changelog |
| Minimum-spec hardware benchmark fails (NFR-009/010) | Medium | High | Benchmarking protocol documented; fallback parameters defined for Argon2id |
| Provider pool does not reach 56 before anticipated launch | High | Critical | Recruit providers through private beta before enabling data owner uploads; FR-053 enforces the gate |
| Data owner loses passphrase and mnemonic simultaneously | Low per user, certain at scale | High | Disclosed clearly at onboarding (FR-005); no support path exists by design |

### 11.4 Rollback Plan

**Microservice:** Replicas are stateless (Postgres is the source of truth). Rollback by
redeploying the previous container image. The audit log is immutable; no data is lost.

**Provider daemon:** The vLog and RocksDB index are on the provider's local disk. A daemon
version rollback does not affect stored data. The new daemon version reads the existing
vLog on startup via crash recovery (FR-056, crash recovery path).

**Payment:** Razorpay payouts that fail return to the master account automatically
(`payout.reversed` webhook). No manual recovery is needed. Idempotency keys prevent
double-payment on retry.

---

## 12. Open Questions

| # | Question | Owner | Due | Linked |
|---|---------|-------|-----|--------|
| OQ-001 | What is the storage rate (paise per GB per month) that makes participation economically viable for providers at MTTF ≥ 180 days while keeping data owner costs below cloud alternatives? | Product | Before private beta | ADR-024, FR-013 |
| OQ-002 | What fraction of Indian home routers are behind symmetric NAT (requiring Circuit Relay v2 permanently)? Global baseline is ~30%; Indian CGNAT prevalence may be higher. | Engineering | Q20-1 — measure at private beta | ADR-021, NFR-006 |
| OQ-003 | Does the dual-window partial hold trigger (0.20 drop in 7d vs 30d score, FR-050) correctly catch degrading providers before the 72h threshold, without penalising providers with legitimate weekend absences? | Engineering | Q31-1 — measure at beta | ADR-024 |
| OQ-004 | What RocksDB rate limiter value satisfies p99 ≤ 200 ms under concurrent compaction on a consumer 7200 RPM HDD? | Engineering | Q27-1 — run benchmark on HDD before launch | ADR-023 |
| OQ-005 | At what provider count N does the observed BWavg exceed 60 Kbps/peer, triggering evaluation of Hitchhiker code adoption for V3? | Engineering | Q39-1 — telemetry at 6 months post-launch | ADR-026 |

---

## 13. Appendix

### 13.1 Requirement Traceability

| ID | Traces to |
|----|-----------|
| FR-001 – FR-006 | DO-01, DO-05, DO-06; ADR-001, ADR-011, ADR-020 |
| FR-007 – FR-014 | DO-01, DO-07; ADR-003, ADR-014, ADR-019, ADR-022 |
| FR-015 – FR-018 | DO-02; ADR-002, ADR-020, ADR-022 |
| FR-019 – FR-021 | DO-03, DO-04; ADR-007, ADR-016 |
| FR-022 – FR-031 | PR-01, PR-02, PR-05; ADR-001, ADR-005, ADR-009, ADR-021, ADR-024, ADR-028 |
| FR-032 – FR-036 | PR-03, PR-07; ADR-007, ADR-024 |
| FR-037 – FR-041 | SY-02; ADR-002, ADR-006, ADR-014, ADR-015, ADR-017, ADR-023, ADR-027 |
| FR-042 – FR-045 | SY-03; ADR-004, ADR-014 |
| FR-046 – FR-052 | PR-04; ADR-011, ADR-012, ADR-016, ADR-024 |
| FR-053 – FR-054 | SY-01; ADR-029 |
| FR-055 – FR-056 | ADR-029 |
| NFR-001 – NFR-003 | ADR-003, ADR-014 |
| NFR-004 – NFR-006 | ADR-003, ADR-021, ADR-025 |
| NFR-007 – NFR-013 | ADR-003, ADR-004, ADR-009, ADR-014, ADR-019, ADR-020, ADR-023 |
| NFR-014 – NFR-020 | ADR-001, ADR-002, ADR-019, ADR-020, ADR-021, ADR-027 |
| NFR-021 – NFR-024 | ADR-015, ADR-016, ADR-023 |
| NFR-025 – NFR-028 | ADR-009, ADR-025; architecture.md §24 |
| NFR-029 – NFR-031 | ADR-011, ADR-012, ADR-024; Paper 35 |

### 13.2 Research Basis

The non-functional requirements in this PRD are not targets chosen arbitrarily. Each derives
from a specific formula, measurement, or proof in the research log:

- **NFR-001** (10⁻¹⁵ loss rate): Giroire Formula 3 applied to s=16, r=40, r0=8, MTTF=300 days
  gives LossRate ≈ 10⁻²⁵ — four orders of magnitude below target.
- **NFR-007** (audit deadline): Derived from the Filecoin Seal timing principle (Paper 29 §3.4.2),
  adapted for timing-based (not cryptographic) outsourcing prevention.
- **NFR-009** (AONT encoding): RFC 8439 (Paper 17) Table B.1 — ChaCha20 at 75 MB/s on OMAP-class
  hardware gives 186 ms per 14 MB segment.
- **NFR-012** (repair bandwidth): Giroire Formula 1 at N=1,000, MTTF=300 days,
  D=50 TB gives BWavg ≈ 39 Kbps/peer.
- **NFR-013** (write amplification): WiscKey (Paper 27) Figure 10 — write amplification ≈ 1.0
  at 256 KB values.

### 13.3 Rejected Requirements

The following were considered and deliberately excluded:

| Rejected requirement | Reason |
|---------------------|--------|
| File deduplication across data owners | Violates zero-knowledge: deduplication requires comparing ciphertexts or plaintexts; both create privacy risks |
| Per-retrieval payment to providers | Swarm SWAP failed structurally (Paper 07). Ties payment to transfer layer, creates liability during microservice outages (ADR-012) |
| Convergent encryption | Explicitly rejected (Paper 16, ADR-022). Each AONT key K is fresh random per segment |
| Blockchain for payment | NBFC registration required; token price volatility; high onboarding friction (ADR-011) |
| Mobile providers in V2 | BWavg at MTTF=90 days ≈ 130 Kbps/peer, exceeding the 100 Kbps budget (ADR-010) |
| Public audit verification in V2 | Requires Transparent Merkle Log — deferred to V3 (ADR-015) |