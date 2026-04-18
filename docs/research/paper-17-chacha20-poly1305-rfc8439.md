# Paper 17 — ChaCha20 and Poly1305 for IETF Protocols

**Authors:** Yoav Nir (Dell EMC), Adam Langley (Google)
**Venue / Year:** IETF RFC 8439 (Informational) | June 2018 | Obsoletes RFC 7539
**Topics:** #9
**ADRs produced:** [ADR-019 — Client-Side Cipher: ChaCha20-256 / AEAD_CHACHA20_POLY1305](../decisions/ADR-019-client-side-encryption.md)

---

## Problem Solved

AES dominates modern encryption but has two practical failure modes: on platforms without
hardware AES acceleration, software AES is three to four times slower than necessary for a
background daemon, and AES table-lookup implementations are vulnerable to cache-collision
timing attacks. This RFC standardises ChaCha20 (a 256-bit stream cipher) and Poly1305 (a
one-time authenticator) as a portable alternative that is constant-time by construction,
fast on any hardware, and produces a combined AEAD primitive (AEAD_CHACHA20_POLY1305) with
a 256-bit key, 96-bit nonce, and 128-bit authentication tag. For Vyomanaut, this document
closes ADR-019 by specifying the cipher primitive for the AONT transform (ADR-022) on
provider hardware that lacks AES-NI — the critical hardware gap for Indian desktop and NAS
providers at the low end of the MTTF range.

---

## Key Findings

**ChaCha20 block function:**
ChaCha20 operates on a 4×4 matrix of 32-bit words. The state is initialised from four
constants, the 256-bit key (8 words), a 32-bit block counter, and a 96-bit nonce (3 words).
Ten iterations of paired column and diagonal quarter-rounds are applied, then the original
state is added back. The output is 64 bytes of keystream. Encryption XORs the keystream
with plaintext. Decryption is identical.

All operations are addition mod 2^32, XOR, and fixed bit-rolls — no S-boxes, no
table lookups. This makes ChaCha20 immune to cache-collision timing attacks and naturally
constant-time in any language that does not branch on key or plaintext values.

**Nonce and counter space:**
- Key: 256 bits
- Nonce: 96 bits — MUST be unique per invocation with the same key
- Block counter: 32 bits — supports up to 2^32 × 64 = 256 GB per (key, nonce) pair
- Nonce reuse with the same key reveals the XOR of two plaintexts — catastrophic

**Poly1305:**
A one-time MAC. Takes a 256-bit one-time key (generated fresh per message) and an arbitrary-
length message, produces a 128-bit tag. The one-time key must never be reused. Security:
forgery probability ≤ 1 - (n / 2^102) for a 16n-byte message, even after 2^64 legitimate
messages (SUF-CMA secure).

**AEAD construction (Section 2.8):**

```
AEAD_CHACHA20_POLY1305(key, nonce, plaintext, aad):
  otk = ChaCha20(key, counter=0, nonce)[0:32]     // one-time Poly1305 key
  ciphertext = ChaCha20(key, counter=1, nonce, plaintext)  // encrypt
  mac_data = aad || pad16(aad)
           || ciphertext || pad16(ciphertext)
           || len(aad) as uint64_le
           || len(ciphertext) as uint64_le
  tag = Poly1305(mac_data, otk)
  return (ciphertext, tag)   // 16-byte tag appended
```

The AEAD construction authenticates both the ciphertext and the associated data (AAD).
Tag comparison MUST use constant-time comparison — memcmp() is not acceptable because
timing differences reveal how long a shared prefix is, reducing forgery search space.

**Performance (Appendix B, measured by Adam Langley):**

| Hardware | AES-128-GCM | ChaCha20-Poly1305 | Winner |
|---|---|---|---|
| OMAP 4460 (ARM, no AES-NI) | 24.1 MB/s | 75.3 MB/s | ChaCha20 3.1× |
| Snapdragon S4 Pro (no AES-NI) | 41.5 MB/s | 130.9 MB/s | ChaCha20 3.2× |
| Sandy Bridge Xeon (AES-NI) | 900 MB/s | 500 MB/s | AES 1.8× |

At 256 KB chunk × 56 = 14 MB per segment, on OMAP-class hardware (stand-in for cheap NAS
or older desktop CPUs):
- AES-128-GCM: ~580 ms per segment
- ChaCha20-Poly1305: ~186 ms per segment

On AES-NI Xeon (modern desktop, fast NAS):
- AES-128-GCM: ~16 ms per segment
- ChaCha20-Poly1305: ~28 ms per segment

For Vyomanaut's ≤5% CPU budget (ADR-009), 186 ms per segment upload is acceptable for
background operation at typical desktop file upload rates.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| ChaCha20 (ARX design) | AES (SPN with table lookups) | Constant-time without hardware support; 3× faster on no-AES-NI hardware; slightly slower on AES-NI hardware |
| 96-bit nonce (IETF variant) | 64-bit nonce (original Bernstein design) | 96-bit is consistent with RFC 5116; limits use to 2^32 blocks (~256 GB) per (key, nonce) — sufficient for 14 MB segments |
| Counter-based nonce management | Random nonces | Eliminates nonce-reuse risk (random nonces have birthday collision at 2^48 invocations); counters are safe but require persistent state |
| Poly1305 as one-time MAC | HMAC-SHA256 as MAC | Poly1305 is faster and constant-time; cannot reuse the key; HMAC can reuse keys |
| AEAD tag = 128 bits (16 bytes) | Truncated tag | RFC explicitly prohibits tag truncation; full 128-bit tag is mandatory |

---

## Breaks in Our Case

- **RFC 8439 requires nonce uniqueness per (key, nonce) invocation** ≠ **the AONT in ADR-022
  generates a fresh random K per segment, so ChaCha20(K, nonce, ...) with a fixed nonce=0
  is safe because K is never reused**
  → For the AONT's internal cipher, K is chosen fresh by SecureRandom(256 bits) per segment.
    With a unique K, the nonce can be fixed at zero without violating the uniqueness
    requirement. Nonce management complexity is zero for this use case.

- **Poly1305 prohibits key reuse** ≠ **the AONT uses SHA-256 for its integrity check, not
  Poly1305**
  → Poly1305 is used for the AEAD construction (pointer file encryption), not inside the AONT.
    Inside the AONT, SHA-256 provides the commitment hash h = SHA-256(c_0 || ... || c_s),
    which is different from a MAC and does not require a one-time key. No conflict.

- **AES-128-GCM is 1.8× faster than ChaCha20-Poly1305 on AES-NI hardware** ≠ **our audit
  challenge response deadline of (chunk_size / declared_upload_speed) × 1.5 is 614 ms for
  a 5 Mbps provider**
  → The cipher choice does not affect the audit deadline — the deadline is about retrieval
    latency, not encryption throughput. Even the slower ChaCha20-Poly1305 path (28 ms per
    14 MB segment on AES-NI) is well within any audit window. This is not a bottleneck.

- **RFC 8439's max message size is 2^32 blocks × 64 bytes ≈ 256 GB** ≠ **our segment size
  is 14 MB**
  → Our 14 MB segment is six orders of magnitude below the 256 GB limit. No constraint applies.

---

## Decisions Influenced

- **[ADR-019](../decisions/ADR-019-client-side-encryption.md) [#9 Client-Side Encryption]** `ACCEPTED`
  Two distinct uses of ciphers in the Vyomanaut pipeline, each with a specified primitive:
  1. **AONT internal cipher:** ChaCha20-256 with nonce=0 (safe because AONT key K is fresh
     per segment). AES-256-CTR on hardware with AES-NI as a performance fast path.
  2. **Pointer file encryption:** AEAD_CHACHA20_POLY1305. The pointer file is the data
     owner's retrieval credential; it must be encrypted before any backup or storage outside
     the owner's device. AAD = file_id + owner_id. Nonce = 96-bit counter, never random.
  *Because:* ChaCha20 is 3× faster than AES on no-AES-NI hardware (OMAP-class, cheap NAS,
  older desktops) and is constant-time without hardware. At 186 ms per segment on the slowest
  target hardware, it fits within the ≤5% CPU background budget (ADR-009). The AEAD
  construction provides authenticated encryption with associated data for the pointer file,
  which needs both confidentiality and integrity.

- **[ADR-022](../decisions/ADR-022-encryption-erasure-order.md) [#15 Encryption-Erasure Interaction]** `UPDATED`
  The AONT cipher is now specified as ChaCha20-256 (soft path) or AES-256-CTR (AES-NI path),
  replacing the generic "AES-256" stated in the ADR. The AONT hash remains SHA-256 —
  Poly1305 is not used inside the AONT.
  *Because:* RFC 8439 Section 4 confirms AES timing attacks are a real concern without
  hardware support. ChaCha20 eliminates this attack surface unconditionally.

---

## Disagreements

- **AES-GCM advocates:** On AES-NI hardware, AES-128-GCM is 1.8× faster than ChaCha20-
  Poly1305. For a datacenter deployment targeting only modern server hardware, AES-GCM would
  be preferred.
  *Implication for us:* Our provider hardware is heterogeneous. Provider daemon must detect
  AES-NI at startup and select the appropriate path. Both paths must be implemented.

- **AONT paper (ADR-022) uses SHA-256 for the hash:** RFC 8439 does not propose replacing
  SHA-256 for general-purpose hashing. BLAKE2b or SHA-3 would be faster alternatives to
  SHA-256 for the AONT hash, but SHA-256 is already in every TLS library and adding a
  second hash library for marginal gain is not warranted.
  *Implication for us:* SHA-256 stays as the AONT hash. RFC 8439 does not change this.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q17-1 through Q17-2.
