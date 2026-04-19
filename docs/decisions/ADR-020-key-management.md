# ADR-020 — Key Management Strategy: Pointer File Management, Capability Model with HKDF Key Hierarchy

**Status:** Accepted
**Topic:** #10 Key Management Strategy
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 18 (Tahoe-LAFS), Paper 16 (AONT-RS), Paper 17 (RFC 8439)

---

## Context

ADR-022 eliminated external AES key storage by embedding the AONT key K inside the coded
data — K is recoverable only when k=16 slices are assembled. This shifted the key management
problem from "how do we store an AES key" to "how do we store and recover the pointer file",
which is now the data owner's sole retrieval credential.

The pointer file contains: the provider list, the chunk IDs for each of the 56 slices, the
erasure parameters, and the file_key (for future re-encryption if the file is updated). If
the pointer file is lost and no backup exists, the file is permanently unrecoverable.

Two questions must be answered:
1. What key hierarchy organises the data owner's credentials so that master secret recovery
   is sufficient to recover all pointer files?
2. What are the structured backup and recovery paths so that device loss does not mean data
   loss?

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Flat per-file keys (one random AES key per file, stored in a local database) | Simple implementation | Loss of local database = loss of all files; no recovery path without full backup |
| Server-held key escrow (microservice stores encrypted versions of all file keys) | Recovery via microservice authentication | Microservice becomes a key management dependency; violates zero-knowledge property if microservice is compromised |
| **HKDF hierarchy with master secret (Tahoe model)** | One master secret recovers all pointer file keys; no key server needed; zero-knowledge preserved | Master secret loss = permanent loss of all files; master secret must be backed up by the data owner |

## Decision

Adopt a three-level key hierarchy derived from a single per-owner master secret.

---

### Level 1 — Master Secret

```
master_secret = Argon2id(
  password   = owner's passphrase or hardware key material,
  salt       = owner_id (UUID, stored in the owner's local key store),
  t=3,       // time cost (iterations)
  m=65536,   // memory cost (64 MB)
  p=4,       // parallelism
  output=32  // 256-bit master secret
)
```

The master secret is never stored on disk. It is derived from the owner's passphrase on
every session start. If the owner uses a hardware key (FIDO2/WebAuthn), the hardware key
output serves as the passphrase input to Argon2id.

**Loss scenario:** If the owner forgets their passphrase and has no backup, all files are
permanently unrecoverable. The owner must be clearly informed of this at account creation.

**Backup path:** The owner may optionally generate a 24-word BIP-39 mnemonic from the
master secret and store it offline (paper, safe deposit box). This is the recovery path for
device loss.

---

### Level 2 — File Key

```
file_key = HKDF-SHA256(
  ikm    = master_secret,
  salt   = owner_id,
  info   = "vyomanaut-file-v1" || file_id,   // domain-separated per-file
  length = 32
)
```

The file_key is derived per file, deterministically from the master secret and the file_id.
Knowing the master secret and the file_id is sufficient to re-derive the file_key — no
separate storage of the file_key is required. The file_key is only stored inside the
(encrypted) pointer file for convenience; it is always re-derivable.

---

### Level 3 — AONT Key K

The AONT key K is not stored. It is embedded in the AONT package and recoverable only by
assembling k=16 of the 56 slices and running the AONT decode. K cannot be derived from the
master secret or file_key — it is fresh random per segment per upload. This is the correct
design: K is a per-upload ephemeral, not a persistent credential.

---

### Pointer File Schema

The pointer file is the read-cap equivalent. It is encrypted with AEAD_CHACHA20_POLY1305
(ADR-019) using a key derived from the master secret:

```
pointer_file_encryption_key = HKDF-SHA256(
  ikm  = master_secret,
  salt = owner_id,
  info = "vyomanaut-pointer-v1" || file_id,
  length = 32
)
```

Plaintext pointer file contents:

```
{
  schema_version:   uint8                     // 1
  file_id:          UUID                      // pseudonymous file handle
  upload_ts:        TIMESTAMPTZ               // when this version was uploaded
  erasure_params: {
    s:              uint8 = 16                // reconstruction threshold
    r:              uint8 = 40                // redundancy fragments
    lf:             uint32 = 262144           // fragment size in bytes (256 KB)
    n:              uint8 = 56                // s + r
  },
  segments: [{
    segment_id:     UUID
    segment_index:  uint32                    // 0-based
    file_key:       bytes[32]                 // for convenience; always re-derivable
    providers:      [56]UUID                  // provider_id for each slice
    chunk_ids:      [56]bytes[32]             // SHA256(slice_content) content address
  }],
  owner_sig:        bytes[64]                 // Ed25519 over all above fields
}
```

The pointer file is signed by the data owner's Ed25519 key (separate from encryption keys,
used for pointer file integrity). The signature proves the pointer file was written by the
owner, not forged by the microservice.

---

### Credential Levels Summary

| Credential | Derivation | Stored | Access granted |
|---|---|---|---|
| master_secret | Argon2id(passphrase, owner_id) | Never on disk | All files |
| file_key | HKDF(master, file_id) | Inside encrypted pointer file (redundant) | One file |
| AONT key K | SecureRandom per segment | Never (embedded in AONT package) | One segment |
| pointer_file_enc_key | HKDF(master, "pointer-v1" \|\| file_id) | Never on disk | One pointer file |

---

### Backup and Recovery Paths

**Scenario 1 — Device loss, passphrase known:**
Owner re-installs the daemon on a new device, enters passphrase. Daemon re-derives
master_secret and all file keys. Downloads pointer files from the coordination microservice
(the microservice stores the encrypted pointer file ciphertext as part of file metadata).
Full recovery.

**Scenario 2 — Device loss, passphrase forgotten, BIP-39 mnemonic available:**
Owner uses the mnemonic to reconstruct master_secret. Same as Scenario 1 from that point.

**Scenario 3 — All credentials lost (no passphrase, no mnemonic):**
Permanent data loss. Files cannot be decrypted. This is the stated risk of zero-knowledge
storage and must be disclosed to the data owner at account creation.

**Scenario 4 — Pointer file lost but master_secret known:**
The microservice stores the encrypted pointer file. Owner re-derives
pointer_file_enc_key from master_secret and file_id and downloads + decrypts the stored
ciphertext. Full recovery. (The microservice never had the decryption key — it stores only
ciphertext.)

---

### What the microservice stores

The coordination microservice stores for each file:

```sql
files (
  file_id          UUID     PRIMARY KEY,
  owner_id         UUID     NOT NULL,
  pointer_ciphertext BYTEA  NOT NULL,   // AEAD_CHACHA20_POLY1305 ciphertext of pointer file
  pointer_nonce    BYTEA(12) NOT NULL,  // 96-bit nonce used for this encryption
  pointer_tag      BYTEA(16) NOT NULL,  // 16-byte Poly1305 authentication tag
  uploaded_at      TIMESTAMPTZ NOT NULL,
  schema_version   SMALLINT NOT NULL DEFAULT 1
)
```

The microservice cannot decrypt pointer_ciphertext — it does not have the owner's master
secret or any derived key. It is a blind store for the owner's benefit. This is the
zero-knowledge property: the service never sees plaintext data or decryption keys.

---

### Pointer file nonce management

The AEAD_CHACHA20_POLY1305 nonce for pointer file encryption is a monotone counter, not
random (per ADR-019 and RFC 8439). The counter is stored in the owner's local key store.
If the local key store is restored from backup with a stale counter (Q17-2), the owner must
increment the counter past the last known value before re-encrypting a pointer file. The
daemon enforces this: it refuses to encrypt with a nonce it has used before.

## Consequences

**Positive:**
- One master secret backs up all files — no per-file key management burden on the data owner
- BIP-39 mnemonic gives a well-understood, offline recovery path
- Microservice is a blind store — zero-knowledge preserved even if the microservice is
  compromised
- File_key re-derivability means pointer file backup is sufficient for full recovery (the
  key inside the pointer file is redundant, not the only copy)
- Ed25519 signature on the pointer file proves ownership and prevents microservice forgery

**Negative / trade-offs:**
- Master secret loss without mnemonic = permanent data loss; this risk must be disclosed
  clearly in onboarding UX
- Argon2id at t=3, m=64MB imposes ~200 ms CPU time per session start on minimum-spec
  hardware — acceptable for interactive login, calibrate empirically
- Nonce counter in local key store adds a persistent local state dependency; losing the
  counter requires manual recovery procedure

**Open constraints:**
- Mnemonic backup UX and the mandatory disclosure of the "no recovery without mnemonic"
  risk must be designed before V2 launch — this is a product and UX decision, not
  a cryptographic one
- Argon2id parameters (t, m, p) must be benchmarked on minimum-spec target hardware
  (Indian entry-level desktop, ~2 GB RAM, dual-core) before launch

## References

- [Paper 18 — Tahoe-LAFS](../research/paper-18-tahoe-lafs.md): capability model; downward derivation hierarchy; zero-knowledge property in production
- [Paper 16 — AONT-RS](../research/paper-16-aont-rs-dispersal.md): AONT key K is not stored; pointer file as retrieval credential
- [Paper 17 — RFC 8439](../research/paper-17-chacha20-poly1305-rfc8439.md): AEAD_CHACHA20_POLY1305 for pointer file encryption; nonce counter requirement
- [ADR-019](ADR-019-client-side-encryption.md): pointer file encryption primitive (AEAD_CHACHA20_POLY1305); file_key derivation via HKDF
- [ADR-022](ADR-022-encryption-erasure-order.md): AONT key K not stored; pointer file is sole retrieval credential
- [ADR-017](ADR-017-audit-receipt-schema.md): Ed25519 signing convention used consistently across all signed artefacts
