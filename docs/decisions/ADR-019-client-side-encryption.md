# ADR-019 — Client-Side Cipher: ChaCha20-256 / AEAD_CHACHA20_POLY1305

**Status:** Accepted
**Topic:** #9 Client-Side Encryption
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 17 (RFC 8439), Paper 16 (AONT-RS)

---

## Context

The AONT-RS encoding pipeline (ADR-022) requires a stream cipher for the AONT transform and
a hash for the integrity commitment. The pointer file (the data owner's sole retrieval
credential after ADR-022 eliminated external AES keys) requires authenticated encryption.
Two cipher choices must be made:

1. The AONT internal cipher — used to encrypt each word of a segment during the AONT pass.
2. The pointer file cipher — used to protect the pointer file before backup or off-device
   storage.

The central constraint is provider hardware: Indian desktop and NAS providers at the low
MTTF bound may not have AES-NI acceleration. On that hardware, AES in software is 24–42
MB/s; ChaCha20 in software is 75–131 MB/s — a 3× improvement that changes whether a 14 MB
segment can be encoded within the ≤5% CPU background budget (ADR-009).

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| AES-256-CTR (AONT) + AES-256-GCM (pointer file) | Single cipher; hardware path is 1.8× faster on AES-NI | 3–4× slower on no-AES-NI hardware; AES table-lookup implementations vulnerable to cache-timing attacks without hardware |
| ChaCha20-256 (AONT) + AES-256-GCM (pointer file) | Constant-time everywhere; fast on all hardware | Two different ciphers in the stack |
| **ChaCha20-256 (AONT) + AEAD_CHACHA20_POLY1305 (pointer file)** | Single cipher family; constant-time; 3× faster on no-AES-NI; full AEAD for pointer file | AES-NI fast path (900 MB/s) is underutilised; two code paths needed for hardware detection |

## Decision

Two cipher specifications for two distinct pipeline stages:

---

### Stage 1 — AONT Internal Cipher

**Primary path (no AES-NI or unknown):** ChaCha20-256.
**Fast path (AES-NI detected at daemon startup):** AES-256-CTR.

The cipher is used as a stream cipher inside the AONT transform. For each word d_i of the
AONT package, the keystream is generated and XORed with the plaintext:

```
// ChaCha20 path (software, no AES-NI):
keystream_block_i = ChaCha20(key=K, counter=i/16, nonce=0x000...0)
c_i = d_i XOR keystream_block_i[word_offset_in_block]

// AES-256-CTR path (hardware, AES-NI detected):
c_i = d_i XOR AES-256-ECB(K, i+1)  // counter mode, i+1 matches AONT-RS spec
```

**Nonce for ChaCha20:** Set to all-zeros. This is safe because the AONT key K is chosen
fresh by `SecureRandom(256 bits)` per segment (ADR-022 Step 2). With a unique K, the nonce
can be fixed without violating RFC 8439's uniqueness requirement — the (key, nonce) pair is
never repeated because K is never repeated.

**Hardware detection:** At daemon startup, detect AES-NI via CPUID instruction (x86) or
equivalent platform capability check. Record in a global constant. Never re-check at
runtime. This is the only branch on hardware type in the entire encoding path.

**Word size and block alignment:** ChaCha20 produces 64-byte (512-bit) keystream blocks.
Each AONT word is 16 bytes. Four words fit in one ChaCha20 block. For a 256 KB chunk with
16-byte words: 16,384 words → 4,096 ChaCha20 blocks. Counter increments per block.

---

### Stage 2 — Pointer File Cipher

**Algorithm:** AEAD_CHACHA20_POLY1305 (RFC 8439 Section 2.8).

The pointer file is the data owner's retrieval credential. It must be encrypted before any
backup, cloud sync, or off-device storage. The AEAD construction provides both
confidentiality and integrity — without the tag, a tampered pointer file cannot be mistaken
for a valid one.

**Pointer file AEAD parameters:**

```
key        = 256-bit key derived from data owner's master secret
             via HKDF(master_secret, salt="vyomanaut-pointer-v1", info=file_id)
nonce      = 96-bit counter, incremented per pointer file write
             (stored adjacent to the ciphertext — nonce is not secret, only unique)
plaintext  = serialised pointer file struct (see ADR-022 for schema)
aad        = owner_id || file_id || schema_version (as little-endian uint8)
             (AAD is authenticated but not encrypted)
output     = ciphertext || 16-byte Poly1305 tag
```

**Nonce management:** Never use random nonces for the pointer file. Use a monotone counter
stored in the owner's local key store, incremented before each write. Nonce reuse with the
same key reveals the XOR of two plaintexts and allows tag forgery — it is catastrophic.

**Tag comparison:** MUST use constant-time comparison. Never use memcmp() or language
default equality for tag verification. Use a library primitive explicitly designed for
constant-time comparison (e.g., libsodium `crypto_verify_16`, or a known-correct
implementation derived from D.J. Bernstein's poly1305-donna).

---

### AONT hash — unchanged from ADR-022

The SHA-256 hash `h = SHA-256(c_0 || c_1 || ... || c_s)` inside the AONT is NOT replaced
by Poly1305. The AONT hash is a commitment, not a MAC — it does not use a secret key and
its role is to bind the AONT key K to all the codewords. SHA-256 is correct for this use.
Poly1305 is a one-time MAC and its one-time-key constraint makes it unsuitable here.

---

### Summary table

| Use | Algorithm | Key | Nonce | Notes |
|---|---|---|---|---|
| AONT word encryption (no AES-NI) | ChaCha20-256 | K (random, per segment) | 0 | K uniqueness makes nonce=0 safe |
| AONT word encryption (AES-NI) | AES-256-CTR | K (random, per segment) | counter=i+1 | Matches AONT-RS spec (Paper 16) |
| AONT integrity hash | SHA-256 | none (commitment) | n/a | Unchanged from ADR-022 |
| Pointer file encryption | AEAD_CHACHA20_POLY1305 | HKDF(master, file_id) | 96-bit counter | 16-byte tag, constant-time verify |

## Consequences

**Positive:**
- ChaCha20 is constant-time on any hardware — eliminates cache-timing attacks without
  requiring AES-NI
- 3× faster than AES on no-AES-NI hardware (75 MB/s vs 24 MB/s on OMAP-class); 14 MB
  segment encodes in ~186 ms on slowest target hardware — within ≤5% CPU budget
- AEAD_CHACHA20_POLY1305 for pointer files provides full authenticated encryption with AAD;
  tampered pointer files fail tag verification before any decryption is attempted
- Single cipher family (ChaCha20/Poly1305) reduces library dependencies for the daemon

**Negative / trade-offs:**
- Two code paths (ChaCha20 and AES-CTR) must be implemented and tested for the AONT
- AES-NI fast path (AES-256-CTR, ~900 MB/s) is significantly faster for high-end providers
  uploading large files — the daemon should prefer it when available
- Nonce counter for pointer file encryption requires persistent, durable storage on the
  owner's device; if the counter is lost (device reset), the owner must rotate the pointer
  file key before re-encrypting

**Open constraints:**
- Master secret derivation and rotation for the pointer file key are not specified here —
  covered by ADR-020 (pointer file and key management, deferred to Phase 3 / Tahoe-LAFS
  reading)
- Multi-sender nonce partitioning (RFC 8439 §2.3 note) is not needed in V2: each data owner
  has one daemon, no concurrent senders sharing a key

## References

- [Paper 17 — RFC 8439](../research/paper-17-chacha20-poly1305-rfc8439.md): ChaCha20 and Poly1305 specification; AEAD construction; performance benchmarks; nonce management requirements
- [Paper 16 — AONT-RS](../research/paper-16-aont-rs-dispersal.md): AONT encoding pipeline; AES-256 as the originally specified AONT cipher (now replaced by ChaCha20-256 on no-AES-NI hardware)
- [ADR-022](ADR-022-encryption-erasure-order.md): AONT-RS encoding pipeline; SHA-256 hash (unchanged); pointer file as the sole retrieval credential
- [ADR-009](ADR-009-background-execution.md): ≤5% CPU budget; 186 ms per segment is within budget
- [ADR-020](ADR-020-key-management.md): pointer file backup and master secret management (Phase 3)
