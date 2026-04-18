# ADR-022 — Encoding Pipeline: AONT-RS (Transform-Then-Code)

**Status:** Accepted
**Topic:** #15 Encryption-Erasure Interaction
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 16 (AONT-RS, Resch & Plank, USENIX FAST 2011), Paper 15 (Szabó et al.)

---

## Context

This decision fixes the encoding pipeline for every segment stored on the network. Two orderings
were under consideration: encrypt-then-code (one AES key per file, stored externally, then RS)
and code-then-encrypt (RS first, then AES per-chunk — 56 keys per file). Paper 15 confirmed
that encrypt-then-code preserves zero-knowledge when coding is delegated. Paper 16 introduces
a third path that supersedes both: the All-or-Nothing Transform applied before RS coding, where
the encryption key is self-embedded in the coded data and recoverable only when k slices are
assembled. No external key storage is needed.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Encrypt-then-code (one AES key per file) | Simple; one key per file; well-understood | External key must be stored securely; key loss = permanent data loss; separate key management infrastructure required (ADR-020) |
| Code-then-encrypt (56 AES keys per file) | Strongest per-shard isolation; independent key per chunk | 56× key management overhead at n=56; no security improvement over AONT-RS per Paper 16; closes Q15-2 against this option |
| **AONT-RS (AONT transform → RS code)** | No external key storage; computational security 2^256; storage ≈ Rabin; production-proven at Cleversafe (20+ installations) | Pointer file becomes sole retrieval credential; per-shard AEAD must come from Merkle audit path, not AONT |

## Decision

Adopt AONT-RS as the segment encoding pipeline. Encoding order:
**AONT transform → systematic Reed-Solomon erasure code.**

**Encoding pipeline (per segment, max 14 MB = 56 × 256 KB):**

```
Input: plaintext segment S

Step 1 — Append canary
  Append a fixed-value 16-byte canary word d_s to S.
  The canary allows corruption detection at decode time.

Step 2 — Choose random key
  K = SecureRandom(256 bits)

Step 3 — Encrypt each word
  For i in 0..s:
    c_i = d_i XOR AES-256(K, i+1)
  (Counter mode: each word encrypted with K under counter i+1.)

Step 4 — Commit the key
  h = SHA-256(c_0 || c_1 || ... || c_s)
  c_{s+1} = K XOR h
  (The key K is now hidden: an attacker needs all c_i to compute h,
   and without h they cannot recover K.)

Step 5 — AONT package
  package = [c_0, c_1, ..., c_s, c_{s+1}]
  (One word larger than the original segment.)

Step 6 — Reed-Solomon dispersal
  Apply systematic RS(s=16, r=40) to the AONT package.
  Output: 56 slices of 256 KB each, stored on 56 distinct providers.
  First 16 slices are the direct AONT package words (systematic property).
```

**Decoding pipeline (given any 16 of 56 slices):**

```
Step 1 — RS reconstruction
  RS decode the k=16 received slices → recover AONT package.

Step 2 — Recover K
  h = SHA-256(c_0 || ... || c_s)
  K = c_{s+1} XOR h

Step 3 — Decrypt
  For i in 0..s:
    d_i = c_i XOR AES-256(K, i+1)

Step 4 — Verify integrity
  Check d_s == canary value.
  If wrong: segment is corrupt; escalate to audit subsystem (ADR-002).
```

**Security guarantee:**

With fewer than k=16 slices, the AONT package cannot be fully assembled. Without the complete
package, h is unknown, so K cannot be computed, so no word of the original data can be
decrypted. An attacker must enumerate up to 2^256 values of K to verify a match — computationally
infeasible at any realistic compute budget. The 20% ASN cap (ADR-014) ensures that no single
correlated provider group can hold more than ~11 of 56 slices, keeping the threshold safe.

**Cipher and hash selection:**

Use AES-256 and SHA-256 (the "secure" AONT configuration). The "fast" configuration from the
paper uses RC4-128 + MD5, which must not be adopted: RC4 is cryptographically broken (RFC 7465,
2015). AES-256 throughput on AES-NI hardware: ~105 MB/s per Paper 16's production benchmark.
On hardware without AES-NI, ChaCha20-Poly1305 (reading-list Phase 1 #12) is the substitute —
its software performance exceeds software AES on ARM and older x86.

**Key management implication:**

External AES keys are eliminated from the system design. The data owner's retrieval credential
is the **pointer file**:

```
pointer_file {
  file_id        UUID           // pseudonymous handle
  segment_list   []{
    segment_id   UUID
    provider_ids [56]UUID       // which providers hold each slice
    chunk_ids    [56]BYTEA(32)  // SHA256 of each slice (content address)
    erasure_params {s:16, r:40, lf:256KB}
  }
  created_at     TIMESTAMPTZ
  owner_sig      BYTEA(64)      // Ed25519 over all above fields
}
```

The pointer file must be kept secure by the data owner. If the pointer file is lost, K cannot
be recovered — there is no out-of-band key backup. ADR-020 (key management) must address
pointer file backup, loss recovery, and delegation.

**Per-shard integrity:**

The canary detects segment-level corruption on full decode. For the ongoing audit subsystem
(ADR-002), challenge-response audits operate on the raw slice: `SHA256(slice_data || challenge_nonce)`.
Providers are challenged on their raw RS slice, not on the decoded AONT package. No change to
ADR-002 is required.

## Consequences

**Positive:**
- External key management infrastructure is eliminated — no per-file keys to store, rotate,
  or lose; ADR-020 scope narrows to pointer file management
- Computational security of 2^256 — functionally equivalent to information-theoretic security
  for any realistic threat model
- Storage overhead ≈ Rabin (nb/k + negligible canary + K⊕h word), 3× less than Shamir
- Proven in production at Cleversafe (10-of-16, 20+ commercial deployments, 1 PB+ internal)
- Canary provides corruption detection at segment decode without a separate MAC

**Negative / trade-offs:**
- Pointer file is the sole retrieval credential — its loss means permanent, unrecoverable data
  loss. ADR-020 must be finalised before V2 launch to address this failure mode.
- AONT's internal AES uses counter mode, not AEAD — per-shard authentication relies on the
  existing Merkle audit mechanism (ADR-002), not the AONT
- AONT processing time on hardware without AES-NI may violate the ≤5% CPU budget for large
  uploads (ADR-009). Benchmark required before shipping: target ≤ 200 ms per 14 MB segment
  on minimum-spec desktop hardware.

**Open constraints:**
- RFC 8439 (ChaCha20-Poly1305, reading-list Phase 1 #12) must be read before ADR-019 is
  finalised — it determines whether AES-256 or ChaCha20-Poly1305 is the default AONT cipher
  for hardware without AES-NI
- ADR-020 must be rewritten from "AES key management" to "pointer file management and recovery"
  before V2 launch

## References

- [Paper 16 — AONT-RS](../research/paper-16-aont-rs-dispersal.md): AONT mechanics; security proof; performance benchmarks; Cleversafe production deployment
- [Paper 15 — Szabó et al.](../research/paper-15-szabo-encrypt-erasure-separation.md): confirms encrypt-then-code preserves zero-knowledge; AONT-RS supersedes this ordering
- [ADR-003](ADR-003-erasure-coding.md): RS parameters (s=16, r=40, lf=256 KB) — unchanged
- [ADR-002](ADR-002-proof-of-storage.md): per-shard Merkle audit — unchanged; challenges raw slices directly
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap ensures threshold security holds against colluding provider groups
- [ADR-019](ADR-019-client-side-encryption.md): AONT = the client-side encryption; cipher primitive TBD pending RFC 8439
- [ADR-020](ADR-020-key-management.md): pointer file backup and loss recovery is now the key management problem
