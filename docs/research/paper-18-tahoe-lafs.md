# Paper 18 — Tahoe: The Least-Authority Filesystem

**Authors:** Zooko Wilcox-O'Hearn, Brian Warner (allmydata.com)
**Venue / Year:** ACM StorageSS 2008 | 6 pages
**Topics:** #9, #10
**ADRs produced:** [ADR-020 — Pointer File Management: Capability Model with HKDF Key Hierarchy](../decisions/ADR-020-key-management.md)

---

## Problem Solved

Distributed storage systems traditionally force a choice between two bad options: trust the
storage servers (they can read and forge data), or manage complex per-file encryption keys
(loss means permanent data loss). Tahoe demonstrates in production that a capability-based
access control model — where possession of a short string is both necessary and sufficient
for access — can be implemented on top of standard cryptographic primitives, eliminating
server trust without creating unmanageable key complexity. At allmydata.com (6.5 million
files, 9.5 TB, 64 servers, 340 concurrent clients), this model ran reliably from April to
October 2008. For Vyomanaut, Tahoe answers ADR-020 directly: it is the only deployed system
that solved zero-knowledge distributed storage with structured key delegation, and its
three-tier capability hierarchy (read-write → read-only → verify) maps exactly onto the
roles that exist in Vyomanaut's provider-audit model.

---

## Key Findings

**Capability model — three tiers for immutable files:**

```
verify-cap  = SHA256d(Capability_Extension_Block)
              → can check integrity, cannot read plaintext

read-cap    = (verify-cap, symmetric_encryption_key)
              → can read and verify, cannot forge

[no write-cap for immutable files — written once only]
```

For immutable files, the read-cap is the data owner's retrieval credential. It is compact
enough to be embedded in a URL. Knowing a read-cap does not reveal the verify-cap alone
because SHA256d is one-way — but the read-cap holder can compute the verify-cap by hashing.
This is called diminishing a capability: always downward, never upward.

**Capability model — three tiers for mutable files:**

```
signing key (SK)    → write-cap derives WK = SHA256d_trunc(SK)
write key (WK)      → read-cap derives RK = SHA256d_trunc(WK)
read key (RK)       → encryption key EK = SHA256d(RK, salt)
verifying key (VK)  → verify-cap VC = SHA256d(VK)
```

The signing key is stored encrypted with WK on the servers. This means the write-cap holder
can recover SK without any separate key backup. WK is short enough (16 bytes) to embed in
a URL. The entire key hierarchy collapses to a single 16-byte root secret.

**The key insight — no separate key management infrastructure:**
The capability IS the key. Whoever has the capability has access. This eliminates the key
distribution problem: to share access, share the capability string. To revoke access,
move the data (re-encrypt and re-upload with a new key). There is no key server, no PKI for
data access, no revocation list.

**Integrity structure — two Merkle trees:**
Tahoe uses two SHA256d Merkle trees per file:
1. A Merkle tree over the erasure code shares — allows identifying which specific share is
   corrupt without downloading all shares.
2. A Merkle tree over the ciphertext — guarantees a one-to-one mapping between verify-cap
   and file contents, preventing a malicious writer from mixing shares from two different
   files.

The roots of both Merkle trees are stored in a "Capability Extension Block" on each server.
The verify-cap is SHA256d(Capability Extension Block).

**Tag-specific hashing:**
Every use of SHA256d is domain-separated: a purpose-specific tag is prepended to the hash
input. This ensures that a hash produced for one purpose (e.g., generating a leaf of a
Merkle tree) cannot collide with a hash produced for another purpose (e.g., generating a
capability string). This is the correct practice for all hash uses in Vyomanaut (ADR-017).

**Freshness for mutable files:**
Each version of a mutable file has a sequence number. The read protocol queries K+E servers
in a first phase, collects sequence numbers, and only proceeds to download when K shares
of the highest sequence number are confirmed available. This makes it hard for a server to
serve a stale version without colluding with enough servers to hold back K shares.

**Production deployment (Section 7):**
- 64 servers, ~340 clients concurrently, ~300 web users per day
- 6.5 million files, ~9.5 TB total
- Launched April 2008, operational with paying customers by May 2008
- RS erasure coding with K=3, N=10 — any 3 of 10 servers sufficient for recovery

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Capability as key (possession = access) | ACL-based access control | No server-side access state to maintain; sharing is just sending a string; revocation requires re-encryption and re-upload |
| Two Merkle trees (shares + ciphertext) | One Merkle tree over all data | Share-level corruption isolation + one-to-one verify-cap → file mapping; slightly more metadata per file |
| WK = SHA256d_trunc(SK) for mutable files | Separate storage of SK | SK recoverable from WK without external key backup; key-dependent encryption is non-standard but reduces key surface |
| Convergent encryption with convergence secret | Random encryption keys | Optional deduplication across users sharing the same secret; security depends on secret staying private |
| K=3, N=10 erasure parameters | Higher redundancy | 70% overhead; low repair cost; suitable for datacenter-class servers with high MTTF |

---

## Breaks in Our Case

- **Tahoe uses AES-128-CTR as the encryption primitive** ≠ **Vyomanaut uses ChaCha20-256
  (ADR-019) for hardware-agnostic constant-time encryption**
  → The capability model and key hierarchy apply unchanged. Only the cipher inside
    `Encrypt(EK, plaintext)` changes. AES-128-CTR is replaced by ChaCha20-256. The AONT-RS
    pipeline (ADR-022) further replaces the simple encrypt-then-code ordering with AONT
    transform, but the capability hierarchy that manages access to the AONT key K is
    identical to Tahoe's read-cap model.

- **Tahoe requires RSA key pair generation per mutable file (0.8–3.2 seconds per file)**
  ≠ **Vyomanaut V2 stores only immutable files (upload-once, no in-place update)**
  → Mutable file complexity (RSA signing keys, write-caps, key-dependent encryption of SK)
    is not needed in V2. The simpler immutable file model (read-cap = verify-cap +
    encryption key) is the correct reference. Key generation time is not a constraint.

- **Tahoe's verify-cap is computed from the Capability Extension Block stored on servers**
  ≠ **Vyomanaut uses the pointer file as the retrieval credential, not a server-stored block**
  → The verify-cap concept maps to Vyomanaut's Merkle challenge mechanism (ADR-002, ADR-015).
    The audit service already holds the chunk hashes. The read-cap equivalent is the pointer
    file. The separation between "can verify integrity" (verify-cap) and "can read plaintext"
    (read-cap) is implemented by the combination of ADR-002 (challenges use chunk hashes,
    not decryption keys) and ADR-019 (AONT key K is never held by providers or auditors).

- **Tahoe's capability strings are short enough to embed in URLs (≤ ~200 chars)**
  ≠ **Vyomanaut's pointer file is a multi-KB structured record**
  → The URL-shortness requirement drove the complex key-dependent encryption of mutable
    files. Vyomanaut does not need URL-embeddable capabilities — the pointer file is a
    structured binary file stored by the data owner. Length is not a constraint. This
    eliminates the non-standard cryptographic constructions from mutable files entirely.

- **Tahoe uses convergent encryption optionally for deduplication**
  ≠ **Vyomanaut explicitly decided not to use convergent encryption (Q09-3, ADR-009)**
  → Deduplication is not a requirement; data owners pay for their own content. Convergent
    encryption is not implemented. The AONT key K is always chosen fresh per segment.

---

## Decisions Influenced

- **[ADR-020](../decisions/ADR-020-key-management.md) [#10 Key Management]** `ACCEPTED`
  Adopt Tahoe's immutable-file capability hierarchy, adapted for Vyomanaut's pointer file
  model. Three credential levels, each derivable downward from the level above:
  1. **master secret** → derived from owner's password or hardware key via Argon2id
  2. **file_key** → HKDF(master_secret, salt="vyomanaut-file-v1", info=file_id)
  3. **AONT key K** → embedded in the AONT package (not stored externally); recoverable
     only by reconstructing k=16 slices
  The pointer file is the read-cap equivalent: it contains file_key (for decrypting future
  versions) and the provider + chunk list needed to assemble the AONT package.
  *Because:* Tahoe proves this hierarchy works in production. The downward-only derivation
  means the data owner needs only the master secret for recovery. No key server is required.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]** `CONFIRMED`
  Tahoe Section 3 describes a Proof of Retrievability protocol as a work in progress —
  "downloading and verifying a randomly chosen subset of the ciphertext." This is exactly
  Vyomanaut's PoR Merkle challenge mechanism. Tahoe validates the approach independently.

- **[ADR-017](../decisions/ADR-017-audit-receipt-schema.md) [#2 Audit Receipt Schema]** `CONFIRMED`
  Section 4.2 mandates domain-separated hashing: every use of SHA256d must prepend a
  purpose-specific tag. Vyomanaut's audit receipt uses `HMAC(server_secret, chunk_id +
  server_ts)` for challenge nonces (ADR-017) — this already implements domain separation.
  The same principle must be applied to all hash uses in the encoding pipeline: the AONT
  hash `h = SHA256(c_0 || ... || c_s)` should prepend a purpose tag to prevent cross-use
  collisions with chunk content hashes.

- **[ADR-019](../decisions/ADR-019-client-side-encryption.md) [#9 Client-Side Encryption]** `CONFIRMED`
  Tahoe's encrypt-then-code ordering and the derivation of encryption key from a short root
  secret via a KDF confirms the Vyomanaut pipeline design. HKDF replaces Tahoe's truncated
  SHA256d for key derivation — HKDF is the standardised, purpose-built KDF primitive.

---

## Disagreements

- **Tahoe uses RSA-2048 for mutable file signing:** The paper itself notes the 0.8–3.2
  second key generation time and proposes ECDSA as a replacement (Section 6.1). For
  Vyomanaut, Ed25519 (already used in ADR-017 for audit receipt signatures) handles
  digital signing at sub-millisecond key generation.
  *Implication for us:* Use Ed25519 throughout — consistent with ADR-017, faster than RSA.

- **Tahoe's K=3, N=10 parameters assume datacenter-class server reliability.** The paper
  was designed for the allmydata.com service with professionally operated servers.
  *Implication for us:* Vyomanaut's K=16, N=56 parameters from ADR-003 are correct for
  our lower-MTTF consumer desktop provider base. Tahoe's parameters are not transferable.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q18-1 and Q18-2.
