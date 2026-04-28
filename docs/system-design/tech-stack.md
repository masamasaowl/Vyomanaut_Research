# Vyomanaut V2 — Technology Stack Rationale

**Status:** Authoritative — update this document whenever a dependency version is pinned
or a technology choice is revisited. Where this document conflicts with an ADR, the ADR
wins.
**Version:** 1.0
**Date:** April 2026
**Author:** Vyomanaut Engineering
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Supersedes:** —
**Companion documents:**
- [`architecture.md`](./architecture.md) — system overview and component descriptions
- [`requirements.md`](./requirements.md) — functional and non-functional requirements
- [`trade-offs.md`](./trade-offs.md) — accepted trade-offs and what was consciously rejected
- [`ADR index`](../decisions/README.md) — all accepted architectural decisions

---

## 1. How to Read This Document

This document records why each technology in the Vyomanaut V2 implementation stack was
chosen over the alternatives considered. It is not a tutorial. It does not explain how to
use the technologies. It explains why they were selected, with concrete performance evidence
from the research log.

Every technology choice here traces to at least one accepted ADR. Where an ADR's
"Options Considered" table lists alternatives, those alternatives appear in the rejection
table within the relevant section below. The research paper citations use the same
numbering as the ADR index.

**Sections 3 through 16** each follow the same format:

1. **Selected technology and version pin** — exact library path and version (or `TBD —
   pin before MN closes` if not yet decided).
2. **Why this over alternatives** — a table of rejected alternatives with reasons.
3. **Performance contract** — a measurable, cited number that justifies the choice.
4. **Lock-in risk** — one sentence on replacement cost; rating used in Section 18.
5. **Source ADR** — hyperlink to the decision record.

**Section 17** lists technologies explicitly rejected to prevent them re-entering future
PRs. **Section 18** summarises lock-in risk across the full stack.

---

## 2. Stack Summary Table

| Layer | Technology | Version pin | Source ADR |
|---|---|---|---|
| Primary language | Go | `≥1.22` — pin before M0 closes | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| Database | PostgreSQL | `≥15` (required for `NULLS NOT DISTINCT`) | [ADR-013](../decisions/ADR-013-consistency-model.md), [ADR-015](../decisions/ADR-015-audit-trail.md) |
| Provider storage index | RocksDB via `linxGnu/grocksdb` | TBD — pin before M2 closes | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| Provider storage values | Custom append-only vLog | — (in-house) | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| P2P framework | `libp2p/go-libp2p` | TBD — pin before M3 closes | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Primary transport | QUIC v1 (RFC 9000) via libp2p | Bundled with go-libp2p | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Fallback transport | TCP + Noise XX + yamux via libp2p | Bundled with go-libp2p | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| AONT cipher (no AES-NI) | ChaCha20-256 via `golang.org/x/crypto` | TBD — pin before M1 closes | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| AONT cipher (AES-NI) | AES-256-CTR via `crypto/aes` (stdlib) | stdlib | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| Authenticated encryption | AEAD_CHACHA20_POLY1305 via `golang.org/x/crypto` | TBD — pin before M1 closes | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| Erasure coding | `klauspost/reedsolomon` | TBD — pin before M1 closes | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Key derivation (operational) | HKDF-SHA256 via `golang.org/x/crypto` | TBD — pin before M1 closes | [ADR-020](../decisions/ADR-020-key-management.md) |
| Key derivation (master secret) | Argon2id via `golang.org/x/crypto` | TBD — pin before M1 closes | [ADR-020](../decisions/ADR-020-key-management.md) |
| Digital signatures | Ed25519 via `crypto/ed25519` (stdlib) | stdlib | [ADR-017](../decisions/ADR-017-audit-receipt-schema.md) |
| Payment gateway | Razorpay Route + Smart Collect 2.0 + RazorpayX | REST API; pin SDK version before M6 closes | [ADR-011](../decisions/ADR-011-escrow-payments.md) |
| Secrets management | HashiCorp Vault / AWS SSM / GCP Secret Manager | Infrastructure decision — any acceptable | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |
| Observability | Prometheus + Grafana | TBD — pin exporter version before M8 closes | [architecture.md §24](./architecture.md#24-observability) |

---

## 3. Language: Go

**Selected:** Go `≥1.22`.

**Why Go over alternatives:**

| Alternative | Why rejected |
|---|---|
| Rust | Memory safety is valuable but Go's goroutine model maps more naturally onto the concurrent challenge-dispatch and upload-pipeline workloads. The engineering team has existing Go fluency. Rust's async ecosystem (Tokio) would require a different concurrency model for the single writer goroutine pattern in [ADR-023](../decisions/ADR-023-provider-storage-engine.md). |
| Python / JavaScript | Interpreted languages cannot satisfy [NFR-009](./requirements.md#73-performance) (AONT encode ≤200 ms per 14 MB segment without AES-NI) or [NFR-008](./requirements.md#73-performance) (audit lookup p99 ≤100 ms on SSD). |
| Java / JVM | JVM warm-up time is incompatible with the daemon auto-start model ([ADR-009](../decisions/ADR-009-background-execution.md)); GC pauses can cause audit RTO violations. |

**Performance contract:** Go's `crypto/chacha20poly1305` implementation achieves
≥75 MB/s on OMAP-class hardware without hardware acceleration
([Paper 17](../research/paper-17-chacha20-poly1305-rfc8439.md) Table B.1), sufficient
to encode a 14 MB segment in ≤186 ms and satisfy [NFR-009](./requirements.md#73-performance).

**Lock-in risk:** Low. The daemon and microservice expose well-defined HTTP and libp2p
interfaces; a language rewrite would not require changing the protocol or data model. Migration
cost: medium (full rewrite, but interfaces are stable).

**Source ADR:** [ADR-001](../decisions/ADR-001-coordination-architecture.md),
[ADR-009](../decisions/ADR-009-background-execution.md).

---

## 4. Database: PostgreSQL

**Selected:** PostgreSQL `≥15` (the `NULLS NOT DISTINCT` syntax in the
`chunk_assignments` uniqueness constraint requires PG 15; see
[data-model.md §4.5](./data-model.md#45-chunk_assignments)).

**Why PostgreSQL over alternatives:**

| Alternative | Why rejected |
|---|---|
| MySQL / MariaDB | Row-level security policies (used to enforce INSERT-only on `audit_receipts` and `escrow_events`) are a PostgreSQL-specific feature. Without them, Invariants 1 and 2 from [data-model.md §3](./data-model.md#3-design-invariants) cannot be enforced at the database layer independently of application code. |
| CockroachDB / distributed SQL | Adds operational complexity with no benefit at V2 scale. The Bailis I-confluence analysis ([ADR-013](../decisions/ADR-013-consistency-model.md)) shows 14 of 20 operations are coordination-free; the 6 coordinated operations go through a single payment service, not through distributed SQL. |
| MongoDB | The append-only CRDT escrow ledger pattern ([ADR-016](../decisions/ADR-016-payment-db-schema.md)) requires reliable INTEGER arithmetic and UNIQUE constraint enforcement on `idempotency_key`. MongoDB's document model and eventual-consistency defaults are mismatched to these requirements. |
| SQLite | Insufficient concurrency for a multi-replica microservice cluster issuing thousands of audit challenge receipts per second at scale. |

**Performance contract:** Postgres single-instance INSERT ceiling is approximately
5,000–10,000 rows/sec under standard workload. At V2 launch (hundreds of providers), the
audit receipt INSERT rate is ~tens of rows/sec — three orders of magnitude below the ceiling.
Monthly partitioning of `audit_receipts` is mandatory from day one to manage the
~1.8 TB/year growth at 56 providers × 50 GB/provider
(see [capacity.md §2.3](./capacity.md#23-audit-receipt-table-growth-in-postgres)).

**Lock-in risk:** Medium. The append-only schema and CRDT ledger design encode
business logic into the database layer (row security policies, UNIQUE constraints,
`EXCLUDE USING GIST`). A migration to a different database requires re-implementing
these invariants at the application layer. Migration cost: high.

**Source ADR:** [ADR-013](../decisions/ADR-013-consistency-model.md),
[ADR-015](../decisions/ADR-015-audit-trail.md),
[ADR-016](../decisions/ADR-016-payment-db-schema.md).

---

## 5. Provider Storage Engine: RocksDB + Custom vLog (WiscKey)

**Selected:** RocksDB via `linxGnu/grocksdb` (CGo bindings) for the chunk index;
a custom append-only value log (vLog) for 256 KB chunk values. This implements the
WiscKey key-value separation architecture from [Paper 27](../research/paper-27-wisckey.md).

**Why this over alternatives:**

| Alternative | Why rejected |
|---|---|
| Standard RocksDB (keys + values together) | Write amplification is 10–14× at 256 KB value sizes ([Paper 26](../research/paper-26-lsm-tree.md)). A provider storing 50 GB of chunks would write 500–700 GB to their storage device — an unacceptable background I/O burden that would breach [NFR-011](./requirements.md#73-performance) (≤5% CPU / background I/O). |
| Flat object store (one file per chunk) | Requires an external lookup table (replicating what RocksDB provides), has no Bloom filter support (audit challenge lookups for unassigned chunks would always require a disk read), and no compaction behaviour to bound background I/O. |
| BoltDB / badger | Both store large values in the LSM tree, producing the same write amplification problem as standard RocksDB at 256 KB. Neither implements WiscKey-style value log separation. |
| LevelDB | Same write amplification issue. No Bloom filter per-key configuration. No `fallocate(FALLOC_FL_PUNCH_HOLE)` support in existing Go wrappers for GC. |

**Performance contract:** WiscKey write amplification is ≈1.0 at 256 KB value sizes
([Paper 27](../research/paper-27-wisckey.md) Figure 10), satisfying [NFR-013](./requirements.md#73-performance).
Bloom filter (10 bits/key, ≈1% false-positive rate) eliminates all disk I/O for audit
challenges on chunks not assigned to this provider. Audit lookup requires exactly one
random disk read: ≈1 ms on SSD, ≈12–15 ms on 7,200 RPM HDD — both within the
p99 thresholds in [NFR-008](./requirements.md#73-performance).

**Lock-in risk:** Medium. The WiscKey pattern is portable (the vLog format is documented
in [ADR-023](../decisions/ADR-023-provider-storage-engine.md) and
[data-model.md §4](./data-model.md)); migrating to a different index library requires
translating the crash recovery tail-scan and GC logic but not the vLog format.
Migration cost: medium.

**Source ADR:** [ADR-023](../decisions/ADR-023-provider-storage-engine.md).

---

## 6. P2P Framework: libp2p

**Selected:** `libp2p/go-libp2p`. Version pin: TBD — pin before M3 closes.
The custom DHT key validator (`/vyomanaut/dht-key/1.0.0`) must be pinned in the
daemon initialisation path and verified to survive every `go-libp2p` upgrade via
`TestDHTKeyValidatorPersists` in CI (see [interface-contracts.md §9](./interface-contracts.md#9-dht-key-contract)).

**Why libp2p over alternatives:**

| Alternative | Why rejected |
|---|---|
| Raw QUIC + custom peer discovery | NAT traversal (AutoNAT, DCUtR, Circuit Relay v2), hole-punching, relay coordination, and cryptographic peer identity would all need to be built from scratch. libp2p has already proven this stack at IPFS and Filecoin scale. |
| gRPC over HTTP/2 | No connection migration (QUIC Connection IDs survive IP changes; HTTP/2 over TCP does not); TCP head-of-line blocking; no built-in NAT traversal. DHCP lease rotation on Indian residential ISPs ([ADR-028](../decisions/ADR-028-provider-heartbeat.md)) makes connection migration a first-class requirement. |
| Custom DHT implementation | S/Kademlia ([Paper 03](../research/paper-03-skademlia.md)) parameters (k=16, α=3, disjoint lookups) are implemented in `go-libp2p-kad-dht`. Re-implementing the DHT would replicate a production-hardened library with no gain. |

**Performance contract:** libp2p's DCUtR hole-punching achieves 70% success across
4.4 million traversal attempts ([Paper 30](../research/paper-30-trautwein-dcutr-nat.md)),
with 97.6% of successful connections succeeding on the first attempt — justifying
`max_hole_punch_retries = 1` per [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md).
Post-v0.5 IPFS (using go-libp2p) achieves median DHT lookup latency of 622 ms
([Paper 20](../research/paper-20-trautwein-ipfs.md)).

**Lock-in risk:** High. The custom HMAC DHT key validator, three-tier NAT traversal
configuration, and 0-RTT policy per protocol ID are all tightly coupled to libp2p
internals. Replacing libp2p requires re-implementing NAT traversal, the Kademlia DHT,
and the custom key validator from scratch. Migration cost: high.

**Source ADR:** [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md),
[ADR-001](../decisions/ADR-001-coordination-architecture.md).

---

## 7. Primary Transport: QUIC v1 (RFC 9000)

**Selected:** QUIC v1 via libp2p's QUIC transport (bundled). The libp2p QUIC transport
implements RFC 9000 and integrates TLS 1.3 natively.

**Why QUIC over alternatives:**

| Alternative | Why rejected |
|---|---|
| TCP (as primary) | TCP head-of-line blocking means a lost packet on one chunk stream stalls all other streams on the same connection. QUIC's independent streams eliminate this for the 56-way parallel shard upload. TCP also does not support connection migration — a provider's IP change during an upload would drop all in-flight streams. |
| HTTP/3 framing on top of QUIC | HTTP/3 framing overhead is not adopted; Vyomanaut uses libp2p's connection and stream layer directly, with its own binary chunk transfer protocol. This avoids HTTP header overhead per stream. |
| WebTransport | Not yet production-stable in go-libp2p at the time of this decision. |

**Performance contract:** QUIC Connection ID migration allows in-flight 256 KB shard
transfers to survive a provider's DHCP lease rotation (common on Indian residential ISPs
with 24-hour lease cycles per [ADR-028](../decisions/ADR-028-provider-heartbeat.md)) with
zero application-layer retry logic. Independent stream delivery eliminates TCP
head-of-line blocking for the 56 parallel upload streams per file segment.

**Lock-in risk:** Low. QUIC is an IETF standard (RFC 9000); any conforming implementation
can replace the libp2p QUIC transport. The fallback TCP path already provides an
alternative at the application protocol level. Migration cost: low.

**Source ADR:** [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md).

---

## 8. Fallback Transport: TCP + Noise XX + yamux

**Selected:** TCP with Noise XX handshake and yamux stream multiplexer, via libp2p's
TCP transport (bundled). Activates automatically when UDP is blocked by a middlebox.

**Why this combination over alternatives:**

| Alternative | Why rejected |
|---|---|
| TLS over TCP | Noise XX provides the same cryptographic properties (mutual authentication, forward secrecy) as TLS but is simpler to implement correctly in a P2P setting where both peers may initiate. libp2p has a production-hardened Noise XX implementation. |
| HTTP/2 multiplexing over TCP | HTTP/2 framing overhead and server-push semantics are unnecessary for the binary chunk transfer protocol. yamux provides the same independent stream delivery with lower overhead. |
| Plain TCP (no multiplexing) | Without yamux, each chunk stream requires a separate TCP connection. At 56 parallel streams per segment, this is 56 simultaneous TCP connections per data owner upload — a significant overhead on both the client and provider. |

**Performance contract:** Empirically, TCP and QUIC achieve statistically identical
NAT hole-punch success rates (~70%) in production measurements
([Paper 30](../research/paper-30-trautwein-dcutr-nat.md) — "TCP = QUIC at ~70%
success; protocol-agnostic"). The fallback carries no reliability penalty relative
to the primary.

**Lock-in risk:** Low. TCP is universal; Noise XX and yamux are replaceable with
TLS 1.3 and HTTP/2 respectively if libp2p is replaced. Migration cost: low.

**Source ADR:** [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md).

---

## 9. AONT Cipher: ChaCha20-256 / AES-256-CTR

**Selected:** Two code paths for the AONT internal cipher, detected at daemon startup via
CPUID and set as a package-level constant — never re-checked at runtime.

- **No AES-NI detected:** ChaCha20-256 via `golang.org/x/crypto/chacha20`.
  Version pin: TBD — pin before M1 closes.
- **AES-NI detected:** AES-256-CTR via `crypto/aes` (Go standard library). No version
  pin required.

**Why ChaCha20-256 (no-AES-NI path) over AES-only:**

| Alternative | Why rejected |
|---|---|
| AES-256-CTR on all hardware | Software AES on hardware without AES-NI achieves 24–42 MB/s; ChaCha20 achieves 75–131 MB/s — a 3× improvement that determines whether a 14 MB segment can be encoded within the ≤5% CPU budget ([ADR-009](../decisions/ADR-009-background-execution.md)). AES table-lookup implementations are also vulnerable to cache-timing attacks without hardware acceleration. |
| RC4-128 + MD5 ("fast" AONT-RS configuration) | RC4 is cryptographically broken (RFC 7465, 2015). The AONT-RS paper ([Paper 16](../research/paper-16-aont-rs-dispersal.md)) explicitly specifies this as the "fast" (insecure) configuration and contrasts it with the "secure" AES-256 + SHA-256 configuration. RC4 must not be adopted. |
| Single cipher (ChaCha20 everywhere, no AES-NI fast path) | AES-256-CTR on AES-NI hardware achieves ≈900 MB/s vs. ChaCha20's ≈75–131 MB/s. High-end providers uploading large files would experience unnecessarily degraded performance. The CPUID detection cost (one check at startup) is negligible. |

**Performance contract:** ChaCha20-256 at 75 MB/s (no-AES-NI, OMAP-class hardware per
[Paper 17](../research/paper-17-chacha20-poly1305-rfc8439.md) Table B.1) encodes a
14 MB segment in ≈186 ms — within the p50 ≤200 ms target in [NFR-009](./requirements.md#73-performance).
AES-256-CTR on AES-NI hardware achieves ≈900 MB/s, giving the same segment in ≈16 ms.

**Lock-in risk:** Low. Both ChaCha20 and AES-256-CTR are IETF-standardised. Any
RFC 8439-compliant ChaCha20 implementation produces identical keystreams; any
AES-256-CTR implementation following the AONT-RS counter convention ([Paper 16](../research/paper-16-aont-rs-dispersal.md))
produces identical output. Migration cost: low.

**Source ADR:** [ADR-019](../decisions/ADR-019-client-side-encryption.md),
[ADR-022](../decisions/ADR-022-encryption-erasure-order.md).

---

## 10. Authenticated Encryption: AEAD_CHACHA20_POLY1305

**Selected:** AEAD_CHACHA20_POLY1305 (RFC 8439 Section 2.8) via
`golang.org/x/crypto/chacha20poly1305`. Version pin: TBD — pin before M1 closes.
Used exclusively for pointer file encryption ([ADR-020](../decisions/ADR-020-key-management.md)).

**Why AEAD_CHACHA20_POLY1305 over alternatives:**

| Alternative | Why rejected |
|---|---|
| AES-256-GCM | AES-256-GCM is 3–4× slower on hardware without AES-NI. The pointer file is encrypted on the data owner's device, which may not have AES-NI. Poly1305 is constant-time on all hardware with no hardware dependency; GCM's GHASH has known constant-time issues in some software implementations. |
| AES-256-GCM (pointer) + ChaCha20 (AONT) | Using two different cipher families increases library dependencies. The single-cipher-family choice (ChaCha20/Poly1305 for both AONT and pointer file) is explicitly identified as a positive consequence in [ADR-019](../decisions/ADR-019-client-side-encryption.md). |
| NaCl secretbox | NaCl secretbox uses XSalsa20-Poly1305. XSalsa20 is not the preferred standard for new systems (RFC 8439 standardises ChaCha20-Poly1305). `golang.org/x/crypto/chacha20poly1305` is a direct standard library; NaCl is a wrapper with additional indirection. |

**Performance contract:** The Poly1305 authentication tag is 16 bytes. Tag verification
uses `crypto/subtle.ConstantTimeCompare`, satisfying [NFR-019](./requirements.md#74-security-and-privacy)
(constant-time comparison mandatory — no timing oracle). A tampered pointer file is
rejected before any decryption is attempted.

**Lock-in risk:** Low. AEAD_CHACHA20_POLY1305 is RFC 8439; any compliant implementation
produces identical ciphertexts and tags. The pointer file nonce (12-byte monotone counter)
and AAD structure (`owner_id || file_id || schema_version`) are documented in
[ADR-019](../decisions/ADR-019-client-side-encryption.md) and can be re-implemented
in any language. Migration cost: low.

**Source ADR:** [ADR-019](../decisions/ADR-019-client-side-encryption.md),
[ADR-020](../decisions/ADR-020-key-management.md).

---

## 11. Erasure Coding: Reed-Solomon via `klauspost/reedsolomon`

**Selected:** `github.com/klauspost/reedsolomon`. Version pin: TBD — pin before M1
closes. Must be vendored if API breaking changes are detected in a minor release.
RS parameters are fixed constants: s=16, r=40, n=56, lf=256 KB
([ADR-003](../decisions/ADR-003-erasure-coding.md)).

**Why `klauspost/reedsolomon` over alternatives:**

| Alternative | Why rejected |
|---|---|
| Clay codes (MSR) | Sub-packetisation at (n=56, k=16): α ≥ 40^16 — computationally intractable ([Paper 22](../research/paper-22-goparaju-msr-codes.md) definitively answers Q19-2). Clay codes are removed as a V3 candidate. |
| LRC (Azure-style) | Non-MDS (extra storage overhead); local group co-locality cannot be guaranteed in a consumer P2P network where providers are behind NAT on different ISPs across India. Repair benefit collapses to RS-level in the worst case ([ADR-026](../decisions/ADR-026-repair-bw-optimisation.md)). |
| Custom GF(2^8) implementation | `klauspost/reedsolomon` is production-hardened with SIMD acceleration (AVX-512, AVX2, NEON) and an extensive test suite. Re-implementing GF arithmetic from scratch introduces correctness risk with no performance benefit. |
| Intel ISA-L via CGo | Additional CGo dependency; `klauspost/reedsolomon` achieves comparable throughput in pure Go with SIMD intrinsics, avoiding the CGo compilation and cross-platform complexity. |

**Performance contract:** `klauspost/reedsolomon` encodes n=56 shards of 256 KB in
significantly less than the 200 ms encoding budget for a 14 MB segment — the library
is CPU-bound on GF arithmetic which is accelerated by SIMD. The Cleversafe production
deployment (10-of-16, 1 PB+ internal, cited in [Paper 16](../research/paper-16-aont-rs-dispersal.md))
validates the AONT-RS pattern at commercial scale. r=40 is the analytically optimal
redundancy level per Giroire Formula 4 (∂BWavg/∂r = 0 at r=40 for s=16, r0=8).

**Lock-in risk:** Low. Reed-Solomon over GF(2^8) is a standard algorithm; any
compliant library produces identical coded data for the same parameters. The systematic
form (first 16 shards are identity-mapped) is an explicit configuration in
`klauspost/reedsolomon`. Migration cost: low.

**Source ADR:** [ADR-003](../decisions/ADR-003-erasure-coding.md),
[ADR-022](../decisions/ADR-022-encryption-erasure-order.md).

---

## 12. Key Derivation: HKDF-SHA256 + Argon2id

**Selected:** Two derivation primitives serving distinct purposes.

- **HKDF-SHA256** via `golang.org/x/crypto/hkdf` — for all operational key derivation
  (file keys, pointer file encryption keys, keystore encryption keys, cluster audit
  secrets). Version pin: TBD — pin before M1 closes.
- **Argon2id** via `golang.org/x/crypto/argon2` — for master secret derivation from
  the data owner's passphrase. Parameters: t=3, m=65536 (64 MB), p=4.
  Version pin: shared with `hkdf` in `golang.org/x/crypto`.

**Why HKDF-SHA256 over alternatives:**

| Alternative | Why rejected |
|---|---|
| PBKDF2 (for operational keys) | PBKDF2 is designed for password-based key derivation (high iteration cost), not for fast deterministic domain-separated key derivation from an already-strong secret. HKDF is the correct primitive for the latter use case (RFC 5869). |
| Direct SHA-256 hashing for key derivation | Raw SHA-256 without the HMAC construction does not provide domain separation and is vulnerable to length-extension attacks. HKDF's `info` parameter provides cryptographic domain separation between (e.g.) file keys and pointer file keys derived from the same master secret. |
| scrypt (for master secret) | Argon2id won the Password Hashing Competition (2015) and is recommended by OWASP over scrypt for new systems. Argon2id's memory-hard property resists both GPU and side-channel attacks. |
| bcrypt (for master secret) | bcrypt has a 72-byte password input limit, which is insufficient for high-entropy passphrases or hardware key material. Argon2id has no such limit. |

**Performance contract:** Argon2id at t=3, m=64 MB, p=4 completes in ≈200 ms
on minimum-spec hardware (per benchmarking protocol Q18-1, [NFR-010](./requirements.md#73-performance):
p50 ≤500 ms target). HKDF-SHA256 is computationally negligible (microseconds per
derivation). The Tahoe-LAFS capability model ([Paper 18](../research/paper-18-tahoe-lafs.md))
validates one-master-secret → all-operational-keys as the correct production pattern.

**Lock-in risk:** Low. HKDF-SHA256 is RFC 5869; Argon2id is RFC 9106. Both are
standardised and implementable in any language from their specifications. The derivation
inputs (domain strings `"vyomanaut-file-v1"`, `"vyomanaut-pointer-v1"`, etc.) are
documented in [ADR-020](../decisions/ADR-020-key-management.md). Migration cost: low.

**Source ADR:** [ADR-020](../decisions/ADR-020-key-management.md),
[ADR-019](../decisions/ADR-019-client-side-encryption.md),
[ADR-027](../decisions/ADR-027-cluster-audit-secret.md).

---

## 13. Digital Signatures: Ed25519

**Selected:** Ed25519 via `crypto/ed25519` (Go standard library). No version pin
required — stdlib.

Ed25519 is used in four places: audit receipt provider signatures, microservice
countersignatures, pointer file owner signatures, and provider heartbeat authentication.
All four use the same key format (`[32]byte` public key, `[64]byte` signature) and the
same verification path.

**Why Ed25519 over alternatives:**

| Alternative | Why rejected |
|---|---|
| ECDSA (P-256) | ECDSA requires a random nonce per signature; nonce reuse is catastrophic (private key recovery). Ed25519 uses a deterministic nonce derived from the private key and message hash — no random number generator is in the signing path. |
| RSA-2048 or RSA-4096 | RSA signatures are 256–512 bytes vs. Ed25519's 64 bytes. At millions of audit receipts per year, the storage difference is significant. RSA key generation is also orders of magnitude slower than Ed25519. |
| BLS signatures (for signature aggregation) | BLS signature aggregation could reduce audit receipt storage if many receipts were batched, but the complexity of BLS is not justified at V2 scale. Ed25519 is simpler to implement correctly and is directly available in the Go standard library. |

**Performance contract:** Ed25519 key generation is sub-millisecond; signing and
verification are also sub-millisecond on any modern hardware. The signature size (64 bytes)
contributes 128 bytes per audit receipt row (provider_sig + service_sig), negligible
relative to the total 297-byte raw row size in [ADR-017](../decisions/ADR-017-audit-receipt-schema.md).

**Lock-in risk:** Low. Ed25519 is standardised (RFC 8032). Keys and signatures are
portable across implementations. Migration cost: low.

**Source ADR:** [ADR-017](../decisions/ADR-017-audit-receipt-schema.md),
[ADR-015](../decisions/ADR-015-audit-trail.md),
[ADR-020](../decisions/ADR-020-key-management.md),
[ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md).

---

## 14. Payment: Razorpay (Route + Smart Collect 2.0 + RazorpayX)

**Selected:** Three Razorpay products used for distinct payment flows, accessed via
the Razorpay REST API. The Go SDK version must be pinned before M6 closes. All payment
logic is isolated behind the `PaymentProvider` interface ([ADR-011](../decisions/ADR-011-escrow-payments.md))
so that the gateway can be replaced without rewriting business logic.

- **Smart Collect 2.0** — inbound UPI virtual accounts for data owner deposits.
- **Route with `on_hold_until`** — programmable settlement hold for the monthly
  escrow release mechanism.
- **RazorpayX Payouts** — outbound bank transfers to provider UPI-linked accounts.

**Why Razorpay over alternatives:**

| Alternative | Why rejected |
|---|---|
| Stripe Connect | Per-transaction fee; settlement takes days vs. UPI's seconds; requires Stripe account (additional onboarding friction for Indian providers). Stripe is the planned international gateway when the `PaymentProvider` interface is extended post-India-launch. |
| Cryptocurrency (Storj token model) | Price volatility; high onboarding friction (crypto wallet required); Swarm SWAP structural failure in production confirmed by [Paper 07](../research/paper-07-sok-dsn.md) (bandwidth-as-currency is structurally incompatible with Vyomanaut's asymmetric model). |
| Razorpay Escrow+ | Requires NBFC registration and trustee approval that Vyomanaut does not qualify for. Route's `on_hold_until` API provides the equivalent hold-and-release primitive without regulatory preconditions ([Paper 35](../research/paper-35-razorpay-upi-docs.md)). |
| Manual bank transfers | No programmatic payout API; no idempotency support; cannot satisfy [FR-047](./requirements.md#610-payment-system) (`X-Payout-Idempotency` header) or [FR-048](./requirements.md#610-payment-system) (automated monthly release). |

**Performance contract:** UPI has zero per-transaction merchant fee and settles in
seconds ([ADR-011](../decisions/ADR-011-escrow-payments.md)). Razorpay Route API rate limit
is 500 requests/minute — sufficient for monthly payout computation up to approximately
120,000 providers in a 4-hour release window (see [capacity.md §6.1](./capacity.md#61-monthly-payout-release-compute)).

**Compliance notes (mandatory, not optional):**
- UPI Collect is deprecated by NPCI as of 28 February 2026. All deposit flows must
  use UPI Intent or QR code ([NFR-029](./requirements.md#77-compliance-and-payments)).
- `X-Payout-Idempotency` header is mandatory on all payout API calls as of
  15 March 2025 ([NFR-030](./requirements.md#77-compliance-and-payments),
  [ADR-012](../decisions/ADR-012-payment-basis.md)).
- RBI bank holiday lookup table must be updated in each December deployment
  ([NFR-031](./requirements.md#77-compliance-and-payments)).

**Lock-in risk:** High. Razorpay's API surface (Smart Collect VPA assignment,
Route transfer IDs, on_hold_until semantics) is embedded in the microservice's
payment flow. The `PaymentProvider` interface insulates the business logic (escrow
computation, release multiplier, seizure trigger) from the gateway; the gateway-specific
code is confined to the Razorpay implementation of that interface. Migrating to a
different gateway (e.g. Stripe Connect for international) requires a new implementation
of `PaymentProvider` plus a data migration of existing Route transfer IDs.
Migration cost: medium.

**Source ADR:** [ADR-011](../decisions/ADR-011-escrow-payments.md),
[ADR-012](../decisions/ADR-012-payment-basis.md),
[ADR-016](../decisions/ADR-016-payment-db-schema.md),
[ADR-024](../decisions/ADR-024-economic-mechanism.md).

---

## 15. Secrets Management: HashiCorp Vault / AWS SSM / GCP Secret Manager

**Selected:** Any of HashiCorp Vault, AWS SSM Parameter Store, or GCP Secret Manager —
the infrastructure team selects based on cloud provider choice. All three are acceptable
provided they support per-path versioning (`/vyomanaut/audit-secret/v{N}`) and
IAM-gated access. The microservice accesses the secrets manager via the
`SecretsManagerClient` interface defined in [interface-contracts.md §8](./interface-contracts.md#8-secrets-manager-contract).

**Why a secrets manager over alternatives:**

| Alternative | Why rejected |
|---|---|
| Environment variables | Environment variables are not versioned, cannot be rotated with a 24-hour overlap window, and are visible in process listings on the host OS. The cluster audit secret rotation protocol ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md) §4) requires versioned paths — environment variables cannot model this without restarting all replicas simultaneously. |
| Per-replica local secret files | A local file cannot be updated across all three replicas atomically. A replica restarting after a crash would load a potentially stale version. No audit trail for secret access. |
| Kubernetes secrets | Acceptable in a Kubernetes deployment. Kubernetes secrets are stored in etcd and distributed to pods. If the deployment uses Kubernetes, this is a valid alternative provided the secret is mounted as a versioned path (not a flat environment variable). |
| `VYOMANAUT_CLUSTER_MASTER_SEED` environment variable | **Permitted only in development and simulation mode.** Presence of this variable in a production deployment is a critical misconfiguration caught by the startup check ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md)). |

**Performance contract:** The secrets manager is not on the hot path. The microservice
caches `server_secret_vN` in memory with a 5-minute TTL ([ADR-027](../decisions/ADR-027-cluster-audit-secret.md)).
Audit challenge generation reads from the in-memory cache — no secrets manager round
trip per challenge. If the secrets manager is unreachable at replica startup, the replica
fails to start (fail-closed — per [ADR-027](../decisions/ADR-027-cluster-audit-secret.md)).

**Lock-in risk:** Low. The `SecretsManagerClient` interface abstracts over all three
providers. Switching from Vault to AWS SSM requires implementing a new 50-line Go
struct, not changing any application logic. Migration cost: low.

**Source ADR:** [ADR-027](../decisions/ADR-027-cluster-audit-secret.md).

---

## 16. Observability: Prometheus + Grafana

**Selected:** Prometheus for metric scraping and alerting; Grafana for dashboards.
Both are deployed as standard cloud-managed or self-hosted instances. Prometheus
exporter version must be pinned before M8 closes. The `prometheus/client_golang`
library version must be pinned alongside.

**Why Prometheus + Grafana over alternatives:**

| Alternative | Why rejected |
|---|---|
| Datadog / New Relic (SaaS APM) | Per-host per-month cost scales with provider count; at 10,000 providers the cost exceeds the observability value. Vyomanaut provider daemons are not directly observable by a SaaS APM without installing an agent on every provider machine, which is not a requirement. |
| OpenTelemetry only (no Prometheus) | OpenTelemetry is a collection standard, not a storage backend. Prometheus is the natural backend for Go's `prometheus/client_golang` pull-based model. OpenTelemetry export to Prometheus is a valid future enhancement but adds indirection at launch. |
| Custom logging-only approach | The operational alerts defined in [architecture.md §24](./architecture.md#24-observability) (repair queue depth, audit TIMEOUT rate, content hash failures, microservice replica count) require time-series data with threshold evaluation — not achievable with structured logging alone. |

**Performance contract:** Prometheus pull-based scraping adds negligible overhead to
the microservice at V2 scale. The metrics defined in [NFR-025](./requirements.md#76-observability-and-operability)
and [NFR-026](./requirements.md#76-observability-and-operability) cover the full
operational surface required to detect the failure modes in [failure-analysis.md](./failure-analysis.md).
The `repair_queue_depth` gauge is the primary early-warning signal for the repair window
exceedance documented in [capacity.md §3.2](./capacity.md#32-burst-repair-bandwidth-qpeek).

**Lock-in risk:** Low. Prometheus metrics format is an open standard; Grafana is open
source. Migrating to a different observability stack (e.g. Grafana Cloud, Victoria
Metrics) requires re-pointing the scrape endpoint, not changing application code.
Migration cost: low.

**Source ADR:** [architecture.md §24](./architecture.md#24-observability),
[requirements.md NFR-025 through NFR-028](./requirements.md#76-observability-and-operability).

---

## 17. Technologies Explicitly Rejected

The following technologies were seriously considered and rejected. This section exists to
prevent them from re-entering the design in future PRs. Any engineer proposing to adopt
a technology on this list must first open a new ADR explaining why the original rejection
reason no longer applies.

| Technology | What it was considered for | Reason rejected | Source |
|---|---|---|---|
| Blockchain / smart contracts | Payment trigger, immutable audit log, public dispute resolution | NBFC registration required for Razorpay Escrow+; token price volatility; high onboarding friction for Indian providers without crypto wallets | [ADR-011](../decisions/ADR-011-escrow-payments.md), [trade-offs.md §Part VII](./trade-offs.md) |
| Razorpay Escrow+ | Native escrow product | Requires NBFC registration and trustee approval that Vyomanaut does not qualify for | [ADR-011](../decisions/ADR-011-escrow-payments.md), [Paper 35](../research/paper-35-razorpay-upi-docs.md) |
| UPI Collect | Data owner deposit flow | Deprecated by NPCI as of 28 February 2026; all deposit flows must use UPI Intent or QR code | [NFR-029](./requirements.md#77-compliance-and-payments), [Paper 35](../research/paper-35-razorpay-upi-docs.md) |
| gRPC over HTTP/2 | P2P chunk transfer | No QUIC connection migration; TCP head-of-line blocking; no built-in NAT traversal; requires stateful server model | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| EigenTrust | Distributed provider reputation scoring | Requires pairwise interaction data between evaluating nodes — data that does not exist when the sole auditor is the microservice; distributed peer audits collapse to universal dishonesty as the unique Nash equilibrium (SHELBY Proposition 1) | [ADR-008](../decisions/ADR-008-reliability-scoring.md), [Paper 24](../research/paper-24-eigentrust.md), [Paper 37](../research/paper-37-shelby-incentive-compatibility.md) |
| Filecoin PoRep + PoSt | Continuous proof of storage | 256 GB RAM + GPU ≥11 GB VRAM required; eliminates every provider in the Indian home desktop target demographic | [ADR-002](../decisions/ADR-002-proof-of-storage.md), [Paper 07](../research/paper-07-sok-dsn.md), [Paper 29](../research/paper-29-filecoin-whitepaper.md) |
| Clay codes (MSR regenerating codes) | Repair bandwidth optimisation (V3) | Sub-packetisation at (n=56, k=16) yields α ≥ 40^16; computationally intractable at these parameters — Q19-2 definitively answered by [Paper 22](../research/paper-22-goparaju-msr-codes.md) | [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md), [Paper 22](../research/paper-22-goparaju-msr-codes.md) |
| LRC (Azure-style locally repairable codes) | Erasure coding | Non-MDS (extra storage overhead); local group co-locality cannot be guaranteed in a consumer P2P network; repair benefit collapses to RS-level under realistic Indian ISP network conditions | [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md), [Paper 19](../research/paper-19-ec-survey.md) |
| Managed consensus (etcd / Consul) | Microservice cluster coordination | External operational dependency; adds latency per coordinated write; overkill for a 3-node cluster where gossip-based membership (Dynamo model) is sufficient | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md), [Paper 12](../research/paper-12-dynamo_1.md) |
| BitSwap (IPFS application protocol) | P2P chunk transfer protocol above the stream | Bandwidth-as-currency is structurally incompatible with Vyomanaut's asymmetric data-owner/provider model; Swarm SWAP failure confirms this is a structural incompatibility, not a parameter calibration issue | [ADR-012](../decisions/ADR-012-payment-basis.md), [Paper 07](../research/paper-07-sok-dsn.md) |
| AES-only ciphers (no ChaCha20 path) | AONT word encryption | Software AES is 24–42 MB/s on no-AES-NI hardware vs. 75–131 MB/s for ChaCha20; cache-timing vulnerability in table-lookup AES without hardware; approximately 10–15% of target Indian desktop providers are estimated to lack AES-NI | [ADR-019](../decisions/ADR-019-client-side-encryption.md), [Paper 17](../research/paper-17-chacha20-poly1305-rfc8439.md) |
| RC4-128 + MD5 ("fast" AONT-RS configuration) | AONT internal cipher | RC4 is cryptographically broken (RFC 7465, 2015). This is the "fast" (explicitly insecure) AONT-RS configuration from [Paper 16](../research/paper-16-aont-rs-dispersal.md). Must never be adopted. | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md), [Paper 16](../research/paper-16-aont-rs-dispersal.md) |
| Mobile providers (iOS / Android) in V2 | Storage providers | MTTF ≈ 90 days → BWavg ≈ 130 Kbps/peer, exceeding the 100 Kbps background bandwidth budget ([Paper 10](../research/paper-10-giroire-lazy.md)); iOS/Android background execution limits kill the daemon unpredictably | [ADR-010](../decisions/ADR-010-desktop-only-v2.md), [Paper 10](../research/paper-10-giroire-lazy.md) |
| Convergent encryption / deduplication | Cross-data-owner deduplication | Requires comparing ciphertexts or plaintexts, creating privacy risks; each AONT key K is fresh random per segment by design — deduplication is structurally incompatible with the zero-knowledge property | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md), [trade-offs.md §Part VII](./trade-offs.md) |
| File versioning / in-place update | Data owner file management | Files are immutable once stored; updating requires delete + re-upload; versioning adds pointer file complexity with no research-validated benefit at V2 scale | [requirements.md §3 Non-Goals](./requirements.md#3-goals-and-non-goals) |
| Coral DSHT | Geographic proximity routing for DHT | Deferred to V3; India-only provider network at V2 has sufficiently homogeneous inter-node latency (5–40 ms between Indian cities) that XOR routing naturally approximates geographic proximity | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |

---

## 18. Technology Lock-in Risk Assessment

| Technology | Lock-in risk | Migration cost | Notes |
|---|---|---|---|
| Go | Low | Medium | Daemon and microservice expose well-defined HTTP and libp2p interfaces; a language rewrite does not require changing the protocol or data model |
| PostgreSQL | Medium | High | Row security policies (INSERT-only invariants), `EXCLUDE USING GIST`, and CRDT ledger pattern encode business logic at the DB layer; must be re-implemented at application layer in any migration |
| RocksDB + vLog | Medium | Medium | WiscKey pattern is portable; vLog format documented in [ADR-023](../decisions/ADR-023-provider-storage-engine.md); GC and crash-recovery code is bespoke but bounded in scope |
| libp2p | High | High | Custom HMAC DHT key validator, three-tier NAT traversal config, and 0-RTT policy per protocol ID are tightly coupled to libp2p internals; NAT traversal must be reimplemented from scratch |
| QUIC v1 | Low | Low | IETF standard (RFC 9000); TCP fallback path already provides an alternative at the application protocol level |
| TCP + Noise XX + yamux | Low | Low | All three are replaceable (TLS 1.3, HTTP/2 multiplexing); the fallback is already a secondary path |
| ChaCha20 / AES-256-CTR | Low | Low | RFC 8439 and NIST standardised; any compliant library produces identical output for the same inputs and counter convention |
| AEAD_CHACHA20_POLY1305 | Low | Low | RFC 8439; nonce and AAD structure documented in [ADR-019](../decisions/ADR-019-client-side-encryption.md); re-implementable from spec |
| `klauspost/reedsolomon` | Low | Low | Standard GF(2^8) RS; replaceable with any library that supports systematic coding at (n=56, k=16) |
| HKDF-SHA256 + Argon2id | Low | Low | RFC 5869 and RFC 9106; domain separation strings documented in [ADR-020](../decisions/ADR-020-key-management.md) |
| Ed25519 | Low | Low | RFC 8032; standard key and signature format; direct Go stdlib support |
| Razorpay | High | Medium | `PaymentProvider` interface insulates business logic; gateway-specific code is confined; migration requires new interface implementation plus data migration of Route transfer IDs |
| Secrets manager | Low | Low | `SecretsManagerClient` interface abstracts over all three providers; switching requires ≈50 lines of Go, not application logic changes |
| Prometheus + Grafana | Low | Low | Open standard metrics format; dashboards and alert rules are configuration files, not application code |

**Overall stack risk profile:** The two highest lock-in risks — libp2p and Razorpay —
are both protected by deliberate interface boundaries. The custom HMAC DHT key validator
and the `PaymentProvider` interface are the critical coupling points. Both are covered
by CI tests (`TestDHTKeyValidatorPersists`, `TestNoFloatArithmetic`) that enforce the
coupling contracts survive dependency upgrades.