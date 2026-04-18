# Paper 16 — AONT-RS: Blending Security and Performance in Dispersed Storage Systems

**Authors:** Jason K. Resch (Cleversafe, Inc.), James S. Plank (University of Tennessee)
**Venue / Year:** USENIX FAST 2011 | 12 pages
**Topics:** #9, #15, #3
**ADRs produced:** [ADR-022 — Encoding Pipeline: AONT-RS (Transform-Then-Code)](../decisions/ADR-022-encryption-erasure-order.md)

---

## Problem Solved

Standard erasure coding (Rabin's IDA) achieves good storage efficiency but zero security: an
attacker with a single slice and knowledge of the generator matrix can verify whether data
matches a known pattern. Shamir's secret sharing fixes this with information-theoretic security
but at n× storage overhead — five times more than Rabin for a 3-of-5 scheme. This paper
introduces AONT-RS, which applies Rivest's All-or-Nothing Transform as a preprocessing pass
before systematic Reed-Solomon coding, achieving computational security equivalent to 2^wA key
enumeration with storage overhead ≈ Rabin's nb/k. For Vyomanaut, this paper closes the
encrypt-then-code vs code-then-encrypt question in ADR-022: AONT-RS is a third path that
subsumes both orderings by replacing separate AES key storage with an embedded-key mechanism
where the key is recoverable only when the reconstruction threshold is met.

---

## Key Findings

**The AONT mechanism:**
Data is divided into s words. A random key K is chosen and each codeword is set to
`c_i = d_i ⊕ AES(K, i+1)`. A final block is `c_{s+1} = K ⊕ SHA256(c_0 || ... || c_s)`.
To recover K, an attacker needs all s+1 codewords — without K, none of the data can be
decrypted. The key is self-contained in the AONT package. No external key storage is required.

**Canary integrity check:**
A known-value word (`canary`) is appended before encoding. On decode, the canary is verified.
If any bit of any stored slice was modified, the SHA-256 avalanche criterion guarantees the
computed hash differs from the original, making the recovered K incorrect, causing the canary
check to fail. This provides segment-level corruption detection without a separate MAC.

**Security level:**
Computational security. With w_A = 256, an attacker who possesses any m < k slices must
enumerate up to 2^256 values of K to verify a data match. The paper calculates that if every
person on Earth ran a trillion computers at a trillion keys per second, the expected time to
break a single key is ~10^35 years — longer than estimates of proton half-life.

**Storage overhead:**
`n(b + w_A) / k` ≈ `nb/k` (Rabin). The extra w_A bytes per segment are negligible at b = 14 MB
(our segment size = 56 × 256 KB). 3× less storage than Shamir at the same security level.

**Performance (single-threaded, 2.8 GHz Xeon):**

| Configuration | Throughput |
|---|---|
| AONTsecure (AES-256 + SHA-256) | 75.6 MB/s (lab), 104.8 MB/s (production) |
| AONTfast (RC4-128 + MD5) | 238–249 MB/s |
| RS dispersal (ce per coding slice) | 966–2628 MB/s |

At k/n ≈ 16/56 ≈ 0.29, RS dispersal cost per segment is low relative to AONT. For n > ~12,
AONT-RS outperforms Rabin. For high k/n rates the crossover happens at smaller n.

**Production validation:**
Cleversafe deployed AONT-RS commercially across 20+ installations in financial, healthcare,
entertainment, and defence sectors. The Museum of Broadcast Communications stores 8,500+ hours
of historical video on 16 servers across 8 sites (Chicago, Dallas, Denver, NJ, SF, Seattle,
Tampa) in a 10-of-16 configuration. The system survived a continental power grid outage with
full data availability. Rolling upgrades are performed without downtime by taking nodes offline
individually — the system remains readable with threshold nodes available throughout.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| AONT embedded key (K ⊕ h in final block) | External AES key storage | No key management infrastructure needed; security comes from the threshold; key loss risk eliminated |
| Systematic RS over non-systematic (Rabin) | Non-systematic dispersal | First k slices are identity-mapped from input — eliminates encoding of those slices; decode only processes non-identity rows; measurable performance gain at high k/n ratios |
| Computational security (2^256) | Information-theoretic security (Shamir) | 3× less storage; same practical security; attacker with infinite compute could break it (but cannot in practice) |
| AES-256 + SHA-256 (secure config) | RC4-128 + MD5 (fast config) | 75 MB/s vs 238 MB/s; paper recommends secure config for production; RC4 is now cryptographically broken and should not be adopted |

---

## Breaks in Our Case

- **AONT-RS provides security without external key storage** ≠ **Vyomanaut requires data owners to
  authenticate retrieval (access control + data recovery)**
  → External encryption keys are eliminated. The data owner's retrieval credential becomes the
    pointer file (provider IDs + chunk IDs + erasure parameters). If the pointer file is lost,
    data is unrecoverable — same risk as losing an AES key, but the attack surface shifts: an
    adversary who steals the pointer file can retrieve but not decrypt without k slices. ADR-020
    (key management) must be updated to cover pointer file backup and loss scenarios.

- **AONT's internal AES uses counter mode (E(K, i+1)), not AEAD** ≠ **our chunk model requires
  per-shard authentication to detect individual slice tampering**
  → The AONT's canary provides segment-level integrity on full decode. Per-shard authentication
    — verifying that a single retrieved slice has not been tampered with — requires the existing
    Merkle challenge mechanism (ADR-002). The canary does not replace per-shard audit; it
    complements it at decode time.

- **Paper benchmarks on a 4-core Xeon at 2.80 GHz with AES-NI** ≠ **our providers include
  heterogeneous desktops and NAS devices, some without AES-NI acceleration**
  → AONTsecure achieves 75–105 MB/s on AES-NI hardware. On a dual-core without AES-NI,
    throughput may drop to 10–30 MB/s. At 256 KB × 56 = 14 MB per segment, AONT pass time on
    slow hardware could reach 500–1400 ms. This must be benchmarked against the ≤5% CPU budget
    (ADR-009). If AES-NI is absent, ChaCha20-Poly1305 (RFC 8439, reading-list Phase 1 #12) is
    the fallback cipher — it is fast in software without hardware acceleration.

- **AONT security requires the attacker cannot acquire all k slices** ≠ **our adversarial model
  includes colluding providers (Honest Geppetto attack)**
  → If an adversary assembles k=16 slices from colluding providers, they recover the AONT
    package and K. The 20% ASN cap (ADR-014) ensures no single correlated group holds more
    than ~11 of 56 slices — well below k=16. Combined with the 4–6 month vetting period
    (ADR-005), the threshold security guarantee holds under our adversarial model.

- **Paper uses RC4-128 for the fast AONT variant** ≠ **RC4 is cryptographically broken (RFC 7465,
  2015)**
  → RC4 must not be adopted. The secure configuration (AES-256 + SHA-256) is the only viable
    option. The fast configuration benchmarks in the paper are informative for understanding
    the performance envelope but the algorithm is not usable in production.

---

## Decisions Influenced

- **[ADR-022](../decisions/ADR-022-encryption-erasure-order.md) [#15 Encryption-Erasure Interaction]** `ACCEPTED`
  Adopt AONT-RS as the segment encoding pipeline. Encoding order: AONT transform → RS erasure
  code. External per-file AES keys are eliminated. The pointer file (provider list + chunk IDs
  + erasure parameters) becomes the sole retrieval credential, managed by the data owner.
  *Because:* AONT-RS achieves computational security equivalent to 2^256 enumeration with
  storage overhead ≈ Rabin. It directly closes Q15-2: code-then-encrypt with per-chunk keys
  offers no meaningful security improvement over AONT-RS for our threat model, at 56× higher
  key management cost. The paper proves that the threshold alone provides sufficient security
  when the key is self-embedded. Production deployment at Cleversafe validates the approach.

- **[ADR-019](../decisions/ADR-019-client-side-encryption.md) [#9 Client-Side Encryption]** `PARTIALLY INFORMED, REMAINS PROPOSED`
  The client-side "encryption scheme" in Vyomanaut is now the AONT (AES-256 + SHA-256). The
  data owner runs the AONT locally before any data leaves the device. No external key is stored.
  However, the paper does not specify an AEAD mode — the AONT's internal AES does not provide
  per-block authentication. ADR-019 should specify the AEAD primitive after RFC 8439 is read
  (reading-list Phase 1 #12), likely ChaCha20-Poly1305 as the internal cipher for
  hardware-agnostic performance.
  *Because:* Paper specifies AES-256 as the cipher but uses ECB-style counter mode, not AEAD.
  AEAD is needed for chunk-level authentication. The cipher choice is not final until RFC 8439
  is read.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `CONFIRMED`
  Systematic Reed-Solomon is correct — AONT-RS is built on systematic RS exactly. The paper
  demonstrates that the systematic property (first k slices are direct AONT package words)
  eliminates encoding overhead for those slices and improves decode performance for partial
  recovery. Parameters s=16, r=40, r0=8, lf=256 KB remain unchanged.
  *Because:* Paper uses systematic RS throughout and directly quantifies why non-systematic
  codes (Rabin) are inferior at high k/n ratios.

- **[ADR-020](../decisions/ADR-020-key-management.md) [#10 Key Management]** `CONSTRAINT CLARIFIED`
  Adopting AONT-RS transforms the problem from "secure storage of AES key" to "secure storage
  of pointer file." The pointer file was already required (it contains provider list and chunk
  IDs). The change is that losing the pointer file now makes data completely unrecoverable with
  no fallback path — K is not stored externally. ADR-020 must address pointer file backup,
  loss recovery, and delegation, not AES key rotation.
  *Because:* AONT's K is derived from the threshold, not stored. The pointer file is the new
  key material.

---

## Disagreements

- **Szabó et al. (Paper 15, FGCS 2025):** Adopts Storj's encrypt-then-code approach as the
  baseline without considering AONT-RS, which predates it by 14 years and achieves the same
  zero-knowledge property with lower key management cost.
  *Implication for us:* AONT-RS is a stronger baseline than pure encrypt-then-code. Szabó's
  traffic reduction findings on mobile access links still apply in V3 but the encryption
  ordering they assume is now superseded.

- **POTSHARDS (Storer et al.):** Uses Shamir's secret sharing for information-theoretic security
  at 3× storage overhead per shard compared to Rabin.
  *Implication for us:* The paper directly shows AONT-RS at 3-of-5 matches Shamir performance
  (65.4 vs 64.4 MB/s) while reducing storage by 3×. At our 16-of-56 configuration, the
  performance advantage of AONT-RS over Shamir is larger.

- **Krawczyk's SSMS (Secret Sharing Made Short, 1993):** Encrypts with a key-based algorithm
  then disperses the key with Shamir. Achieves similar computational security to AONT-RS.
  *Implication for us:* SSMS requires storing and distributing the Shamir shares of the key.
  AONT-RS eliminates this entirely — the key is encoded in the AONT package. AONT-RS is
  simpler for our single-owner data model.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q16-1 through Q16-2.
