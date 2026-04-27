# File Upload

**ADR references:** [ADR-001](../../decisions/ADR-001-coordination-architecture.md) · [ADR-003](../../decisions/ADR-003-erasure-coding.md) · [ADR-005](../../decisions/ADR-005-peer-selection.md) · [ADR-009](../../decisions/ADR-009-background-execution.md) · [ADR-014](../../decisions/ADR-014-adversarial-defences.md) · [ADR-019](../../decisions/ADR-019-client-side-encryption.md) · [ADR-020](../../decisions/ADR-020-key-management.md) · [ADR-021](../../decisions/ADR-021-p2p-transfer-protocol.md) · [ADR-022](../../decisions/ADR-022-encryption-erasure-order.md) · [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) · [ADR-029](../../decisions/ADR-029-bootstrap-minimum-viable-network.md)  
**Architecture reference:** [architecture.md §10 Data Encoding Pipeline](../architecture.md#10-data-encoding-pipeline) · [architecture.md §22 Runtime Flows — Flow 1](../architecture.md#22-runtime-flows)

---

## Overview

File data never flows through the microservice. The microservice is control-plane only: it assigns providers and stores an encrypted pointer file ciphertext it cannot decrypt. All 56 × n_segments shard transfers happen over direct libp2p/QUIC P2P connections between the data owner client and providers ([ADR-001](../../decisions/ADR-001-coordination-architecture.md)).

---

## Phase 1 — Local Encoding Pipeline

```
Data Owner Client (local, no network)
────────────────────────────────────────────────────────────────────────────
     │
     │  INPUT: plaintext file at path/to/file
     │
     ├─ Step 0: Master secret derivation
     │   master_secret = Argon2id(
     │     password   = owner_passphrase,
     │     salt       = owner_id,
     │     t=3, m=65536, p=4,
     │     output=32 bytes
     │   )
     │   (ADR-020; never written to disk, never transmitted)
     │
     ├─ Step 1: Segmentation
     │   IF file_size > 14 MB (56 × 256 KB):
     │     split into ceil(file_size / 14 MB) segments
     │   ELSE:
     │     pad to minimum segment size (4 MB = 16 × 256 KB)
     │     record original_size_bytes for post-decode strip
     │   (ADR-022)
     │
     │  FOR EACH SEGMENT:
     │
     ├─ Step 2: AONT key generation
     │   K = SecureRandom(256 bits)      ← fresh per segment, never stored
     │   (ADR-022; K is embedded in the erasure-coded data)
     │
     ├─ Step 3: AONT transform
     │   detect AES-NI via CPUID at daemon start:
     │
     │   IF no AES-NI (primary path, most Indian desktops):
     │     ChaCha20-256 path (ADR-019):
     │     for i, word_i in enumerate(16-byte words of segment + canary):
     │       keystream_block = ChaCha20(key=K, counter=i/4, nonce=0x00...0)
     │       c_i = word_i XOR keystream_block[word_offset]
     │       ↑ nonce=0 is safe because K is fresh per segment (RFC 8439)
     │
     │   IF AES-NI detected (fast path):
     │     AES-256-CTR path (ADR-019):
     │     c_i = word_i XOR AES-256-ECB(K, i+1)   ← counter mode
     │
     │   h = SHA-256(c_0 ‖ c_1 ‖ … ‖ c_s)          ← commitment hash
     │   c_{s+1} = K XOR h                           ← K embedded in data
     │   AONT_package = [c_0, c_1, …, c_s, c_{s+1}]
     │
     │   Security guarantee: without all s+1 codewords an attacker
     │   cannot compute h, cannot recover K, cannot decrypt any word.
     │   (ADR-022)
     │
     ├─ Step 4: Reed-Solomon dispersal
     │   RS(s=16, r=40) → 56 shards of exactly 256 KB each
     │   shards 0–15:  systematic (direct AONT codewords)
     │   shards 16–55: parity
     │   (ADR-003; r=40 is analytically optimal per Giroire Formula 4)
     │
     ├─ Step 5: Content addressing
     │   chunk_id[i] = SHA-256(shard[i])   for i in 0..55
     │   (used as RocksDB key on provider side, ADR-023)
     │
```

---

## Phase 2 — Assignment Request

```
Data Owner Client               Microservice                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                               │                               │
     │── POST /api/v1/upload/assign ─►│                               │
     │   Authorization: Bearer <JWT>  │                               │
     │   {                            │                               │
     │     file_id: "<UUIDv7>",       │                               │
     │     num_segments: N            │                               │
     │   }                            │                               │
     │                                │                               │
     │                                ├─ READINESS GATE CHECK ────────►│
     │                                │  all 7 conditions satisfied?  │
     │                                │  (ADR-029)                    │
     │                                │                               │
     │   FAILURE PATH:                │                               │
     │   gate not satisfied           │                               │
     │◄── 503 NETWORK_NOT_READY ──────│                               │
     │    { readiness_url: "..." }    │                               │
     │                                │                               │
     │                                ├─ ESCROW CHECK ────────────────►│
     │                                │  balance ≥ cost_30d(file)?    │
     │                                │                               │
     │   FAILURE PATH:                │                               │
     │   escrow balance insufficient  │                               │
     │◄── 409 INSUFFICIENT_ESCROW ────│                               │
     │                                │                               │
     │                                ├─ FOR EACH SEGMENT ────────────►│
     │                                │  SELECT 56 distinct providers │
     │                                │  using Power of Two Choices   │
     │                                │  weighted by reliability score│
     │                                │  (ADR-005)                    │
     │                                │                               │
     │                                │  ENFORCE 20% ASN CAP:         │
     │                                │  for each candidate provider, │
     │                                │  count shards already assigned│
     │                                │  to that provider's ASN in    │
     │                                │  this segment.                │
     │                                │  floor(0.20 × 56) = 11 max   │
     │                                │  per ASN. Skip if exceeded.   │
     │                                │  (ADR-014, ADR-003)           │
     │                                │                               │
     │                                │  FAILURE PATH:                │
     │                                │  ASN cap cannot be satisfied  │
     │                                │  (too few ASNs in pool)       │
     │◄── 503 NETWORK_NOT_READY ──────│                               │
     │                                │                               │
     │                                ├─ INSERT segments ─────────────►│
     │                                │  INSERT chunk_assignments     │
     │                                │  (56 × N rows)               │
     │                                │  status = ACTIVE              │
     │                                │                               │
     │◄── 200 {                   ────│                               │
     │     assignments: [             │                               │
     │       {                        │                               │
     │         segment_index: 0,      │                               │
     │         segment_id: "<UUID>",  │                               │
     │         providers: [           │                               │
     │           {                    │                               │
     │             shard_index: 0,    │                               │
     │             provider_id: "...",│                               │
     │             multiaddrs: [...], │                               │
     │             asn: "AS24560"     │                               │
     │           },                   │                               │
     │           … × 55 more          │                               │
     │         ]                      │                               │
     │       },                       │                               │
     │       … × (N-1) more segments  │                               │
     │     ]                          │                               │
     │   }                            │                               │
     │                                │                               │
```

---

## Phase 3 — Shard Upload to Providers (P2P, no microservice in path)

```
Data Owner Client        Provider Daemon (×56 per segment, parallel)
────────────────────────────────────────────────────────────────────────────
     │                             │
     │  FOR EACH of 56 providers   │
     │  (all in parallel):         │
     │                             │
     ├─ libp2p dial ──────────────►│
     │  using multiaddrs from      │
     │  assignment response        │
     │                             │
     │  NAT traversal (ADR-021):   │
     │  1. try QUIC v1 direct      │
     │  2. DCUtR hole-punch        │
     │     if cone NAT             │
     │     (max_retries = 1)       │
     │  3. Circuit Relay v2        │
     │     if symmetric NAT        │
     │                             │
     │  TLS 1.3 handshake via      │
     │  QUIC (or Noise XX via TCP) │
     │  peer identity verified:    │
     │  peer_id = multihash(pubkey)│
     │                             │
     │── CHUNK_UPLOAD stream ──────►│
     │   { chunk_id,               │
     │     shard_data: 256 KB }    │
     │                             │  provider stores to vLog:
     │                             │  (ADR-023, WiscKey)
     │                             │
     │                             ├─ append vLog entry:
     │                             │   chunk_id     (32 B)
     │                             │   chunk_size   (4 B)
     │                             │   chunk_data   (262,144 B)
     │                             │   content_hash (32 B)
     │                             │     = SHA-256(chunk_data)
     │                             │
     │                             │  SINGLE WRITER GOROUTINE:
     │                             │  (all uploads serialised
     │                             │   through one goroutine,
     │                             │   ADR-023 concurrent write
     │                             │   safety)
     │                             │
     │                             ├─ fsync() vLog
     │                             ├─ INSERT RocksDB:
     │                             │   chunk_id → (vlog_offset,
     │                             │               chunk_size)
     │                             │
     │                             │  FAILURE PATH:
     │                             │  fsync() fails:
     │◄── CHUNK_UPLOAD error ───────│  provider returns error
     │    client retries to        │  client dials next candidate
     │    alternate provider       │  from provider pool
     │    (assignment service      │
     │     can provide alternates) │
     │                             │
     │◄── signed upload receipt ───│
     │    {                        │
     │      chunk_id: "...",       │
     │      provider_id: "...",    │
     │      received_at: "...",    │
     │      provider_sig: "<Ed25519 over all fields>"
     │    }                        │
     │                             │
     │  client stores receipt      │
     │  in session state           │
     │  (crash-safe, FR-060)       │
     │                             │
```

---

## Phase 4 — Pointer File Creation and Registration

```
Data Owner Client               Microservice                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                               │                               │
     │  After all 56 × N receipts    │                               │
     │  collected:                   │                               │
     │                               │                               │
     ├─ Build pointer file struct:   │                               │
     │   {                           │                               │
     │     schema_version: 1,        │                               │
     │     file_id: "<UUID>",        │                               │
     │     original_size_bytes: N,   │                               │
     │     segments: [{              │                               │
     │       segment_id,             │                               │
     │       segment_index,          │                               │
     │       provider_ids: [56],     │                               │
     │       chunk_ids: [56],        │                               │
     │       erasure_params: {       │                               │
     │         s:16, r:40, n:56,     │                               │
     │         lf_bytes:262144 }     │                               │
     │     }],                       │                               │
     │     owner_sig: "<Ed25519 over all above fields>"              │
     │   }                           │                               │
     │                               │                               │
     ├─ Derive pointer encryption key:                               │
     │   pointer_enc_key = HKDF-SHA256(                              │
     │     ikm  = master_secret,     │                               │
     │     salt = owner_id,          │                               │
     │     info = "vyomanaut-pointer-v1" ‖ file_id,                  │
     │     len  = 32 bytes           │                               │
     │   )                           │                               │
     │   (ADR-020; key never transmitted)                            │
     │                               │                               │
     ├─ Encrypt pointer file:        │                               │
     │   nonce = monotone counter    │                               │
     │           from local keystore │                               │
     │           (96-bit, ADR-019)   │                               │
     │   aad   = owner_id ‖ file_id ‖ schema_version                │
     │   {ciphertext, tag} =         │                               │
     │     AEAD_CHACHA20_POLY1305(   │                               │
     │       key   = pointer_enc_key,│                               │
     │       nonce = nonce,          │                               │
     │       aad   = aad,            │                               │
     │       plain = pointer_file    │                               │
     │     )                         │                               │
     │   (ADR-019; tag is 16 bytes)  │                               │
     │                               │                               │
     │── POST /api/v1/file/register ─►│                               │
     │   {                            │                               │
     │     file_id,                   │                               │
     │     pointer_ciphertext: <base64>,                              │
     │     pointer_nonce: <base64>,   │                               │
     │     pointer_tag: <base64>,     │                               │
     │     original_size_bytes,       │                               │
     │     schema_version: 1,         │                               │
     │     owner_sig: "<sig>"         │                               │
     │   }                            │                               │
     │                                │── INSERT files ───────────────►│
     │                                │   pointer_ciphertext stored   │
     │                                │   BLIND: microservice cannot  │
     │                                │   decrypt (ADR-020, NFR-014)  │
     │                                │                               │
     │◄── 201 { file_id, uploaded_at } ──────────────────────────────│
     │                                │                               │
     │   client clears session state  │                               │
     │   (upload complete, FR-060)    │                               │
     │                                │                               │
```

---

## Crash Recovery (Upload Resume)

If the client crashes after some shards are uploaded but before pointer file registration, the local session state (persisted per FR-060) allows the upload to resume without re-transmitting already-acknowledged shards.

```
Data Owner Client (after restart)       Microservice
────────────────────────────────────────────────────────────────────────────
     │                                        │
     │  on restart: scan session state files  │
     │  find incomplete upload for file_id    │
     │                                        │
     │  POST /api/v1/upload/assign (idempotent)
     │  { file_id: "<same UUID>",             │
     │    num_segments: N }                   │
     │                                        │
     │◄── 200 same assignments ───────────────│
     │    (idempotent: returns existing       │
     │     chunk_assignments rows)            │
     │                                        │
     │  for each segment:                     │
     │    for each shard:                     │
     │      IF receipt already in session:    │
     │        skip (already uploaded)         │
     │      ELSE:                             │
     │        re-dial provider, re-upload     │
     │                                        │
     │  continue from where crash occurred    │
     │                                        │
```

---

## Key Invariants

| Invariant | Source |
|---|---|
| File data never flows through the microservice | [ADR-001](../../decisions/ADR-001-coordination-architecture.md), data/control plane separation |
| AONT key K is generated fresh per segment and never stored or transmitted | [ADR-022](../../decisions/ADR-022-encryption-erasure-order.md) |
| 20% ASN cap enforced at assignment time, not retrospectively | [ADR-014](../../decisions/ADR-014-adversarial-defences.md), [ADR-003](../../decisions/ADR-003-erasure-coding.md) |
| Pointer file ciphertext stored by microservice is undecryptable without master_secret | [ADR-020](../../decisions/ADR-020-key-management.md) |
| vLog fsync() before RocksDB INSERT (crash safety) | [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) |
| Single writer goroutine for all vLog appends | [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) |
| Upload session state persisted locally; resume without re-transmitting acked shards | FR-060 |
| Escrow balance must cover 30 days before upload is permitted | FR-014 |