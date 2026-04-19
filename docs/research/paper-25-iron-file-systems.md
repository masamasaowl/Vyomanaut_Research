# Paper 25 — IRON File Systems

**Authors:** Vijayan Prabhakaran, Lakshmi N. Bairavasundaram, Nitin Agrawal, Haryadi S. Gunawi, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau — University of Wisconsin, Madison
**Venue / Year:** SOSP 2005 | Brighton, UK
**Topics:** #16, #19, #2
**ADRs produced:** none — ADR-002 and ADR-004 confirmed; ADR-023 gains new constraints; see Decisions Influenced

---

## Problem Solved

Commodity file systems assume disks either work perfectly or fail completely, yet modern disks exhibit a richer failure spectrum: individual blocks can become inaccessible (latent sector errors) or silently return wrong data (block corruption), without any whole-disk failure signal. The paper names this the *fail-partial* failure model, develops a taxonomy of detection and recovery techniques (IRON), audits four commodity file systems against it, and builds a prototype IRON ext3 (ixt3) that detects corruption via checksums, protects metadata with replication, and protects user data with parity — at under 10% time overhead for typical workloads. For Vyomanaut, the paper's central value is establishing that disk corruption is silent by default, and that the only reliable protection is end-to-end checksumming above the disk layer — which is precisely what the PoR challenge already provides.

---

## Key Findings

**Fail-partial taxonomy — three failure modes (Section 2.3):**
Entire disk failure (classic fail-stop), block failure (latent sector error — block inaccessible), and block corruption (block accessible but data silently wrong). Corruption is the most dangerous because no error code is returned; the storage subsystem simply hands back bad data. Latent sector errors are estimated to occur five times more often than absolute disk failure.

**Specific failure classes relevant to Vyomanaut (Section 2.2):**
Phantom writes — the disk reports a write as successful but the data never reaches media. A provider storing a chunk receives a success return code; the chunk does not exist on disk. Misdirected writes — correct data written to the wrong sector. A provider storing chunk A accidentally overwrites chunk B. Both are now unavailable. Bit rot — individual bits flip over time, silently corrupting data in place.

**IRON taxonomy (Tables 1 and 2):**
Detection levels: DZero (none), DErrorCode (check return codes), DSanity (structural consistency checks), DRedundancy (checksums, replication). Recovery levels: RZero through RRedundancy. The paper's key finding is that commodity file systems (ext3, ReiserFS, JFS, NTFS) universally implement DZero for write errors — they ignore write failure return codes — making corruption from phantom writes invisible.

**Why checksums must be stored separately from the data they protect (Section 3.1):**
A checksum stored adjacent to the block it checksums cannot detect a misdirected write: the wrong block arrives with its own correct checksum, and the check passes. Checksums must be stored at a location derived independently from the block address. Vyomanaut's chunk content address (`SHA256(chunk_data)`) stored in the pointer file is exactly this: it is derived from content, not location, and is held by the microservice independently of the provider's disk.

**ixt3 overhead is modest (Table 6):**
Full IRON protection (metadata checksums, metadata replication, data parity, transactional checksums) adds at most 6% overhead for typical workloads (SSH-Build, web server). Metadata-intensive workloads (PostMark: up to 37%, TPC-B: up to 42%) show higher overhead, but these represent worst-case write-heavy workloads not representative of cold storage access patterns.

**Transactional checksums improve performance (Table 6, row 5):**
By placing a checksum over journal transaction contents in the commit block, ixt3 avoids ext3's extra disk rotation wait before commit, improving TPC-B throughput by 20%.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| End-to-end checksumming (file system level) | Trusting disk-level ECC alone | Detects corruption from all layers above media, including buggy controllers and firmware; adds per-block storage and compute overhead |
| Checksums stored separately from data | Checksums co-located with data | Detects misdirected writes; requires extra storage metadata tracking |
| Lazy detection (on block access) + eager scrubbing (background) | Lazy detection alone | Proactive discovery of latent sector errors before a read attempt fails; scrubbing adds I/O load |
| Metadata replication on distant disk regions | Metadata stored once | Survives spatially-local failures (surface scratches covering contiguous blocks); copies must be spread across platters or distant disk regions |
| Parity per file (one parity block per file) | No user-data redundancy | Recovers one data-block failure per file; incurs 3–17% space overhead depending on file distribution |

---

## Breaks in Our Case

- **Paper models a single local file system on a single disk** ≠ **Vyomanaut is a distributed system where chunks are spread across n=56 providers with independent disks**
→ The fail-partial failure model applies per-provider: each provider's disk can exhibit latent sector errors and block corruption independently. A single provider losing a chunk to disk corruption has the same observable effect as that provider going offline — they fail the PoR audit. The distributed redundancy (RS(16,56)) absorbs single-provider corruption without data loss, exactly as it absorbs single-provider departure.
- **IRON focuses on file system detection and recovery within a single machine** ≠ **Vyomanaut's detection is at the network level via PoR audit challenges**
→ Vyomanaut's PoR challenge (`SHA256(chunk_data || challenge_nonce)`) is the end-to-end checksum the paper advocates. If a provider's disk returns corrupt chunk data, the computed response hash will not match the expected value, and the audit will FAIL — correctly. The audit system detects corruption without any in-daemon disk scrubbing. This is a stronger guarantee than IRON: Vyomanaut's checksum is computed over the actual content, not just the file system metadata.
- **IRON's proactive scrubbing requires a local replica for repair** ≠ **our repair requires contacting k=16 other providers**
→ If the provider daemon scrubs a stored chunk and detects corruption, it cannot self-repair (it holds only one shard of a 16-of-56 scheme). Detection is still valuable: the daemon can proactively report the corruption to the microservice, which queues repair faster than waiting for the next scheduled audit challenge.
- **Phantom writes are invisible at the file system level — a write that never reached disk appears successful** ≠ **our first PoR challenge on a newly stored chunk would immediately detect this**
→ A phantom write during chunk storage means the provider will fail the very first audit challenge. The audit system catches this within one audit cycle (24h), triggering repair. No special in-daemon detection is needed for this case — the distributed audit subsystem handles it.
- **Spatial locality of disk failure: a surface scratch can corrupt many contiguous blocks** ≠ **if the provider storage engine stores multiple chunks in contiguous disk regions, one scratch could fail multiple chunks simultaneously**
→ This is a within-provider correlated failure that ADR-014's ASN cap does not address (the cap targets across-provider correlation). If a provider stores all their chunks contiguously on disk, a single media scratch could fail multiple chunks at once, producing a burst failure event. The provider storage engine (ADR-023) must address this with randomised or spread placement.

---

## Decisions Influenced

- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
The PoR Merkle challenge (`SHA256(chunk_data || challenge_nonce)`) is the end-to-end checksum the IRON paper identifies as the only reliable detection mechanism for corruption from all storage stack layers. The paper's Section 3.1 establishes that checksums stored independently from data (as in our chunk content address in the pointer file) are specifically required to detect misdirected writes — exactly the property our content-addressed chunk IDs provide. The audit mechanism correctly fails any provider whose disk has silently corrupted a chunk.
*Because:* Section 3.4 states the end-to-end argument — the file system (or in our case, the audit layer) is the only place that can detect corruption from layers above the media, including buggy controllers and firmware.
- **ADR-004 [#4 Repair Protocol]** `CONFIRMED`
Physical disk corruption is indistinguishable from intentional data deletion at the audit level: both produce an audit FAIL or TIMEOUT. The lazy repair mechanism (trigger at r0=8) handles corrupt providers identically to departed providers. No special path is needed for hardware failure vs. adversarial deletion.
*Because:* The fail-partial model shows that block corruption and block unavailability are the two sub-cases of partial failure. Both produce the same observable outcome from the audit layer's perspective: the chunk data is unavailable.
- **ADR-023 [#16 Provider Storage Engine]** `NEW CONSTRAINTS`
This paper adds two constraints that the ADR-023 decision must satisfy:
(1) **Per-chunk content hash stored and verified on read.** The provider daemon must store `SHA256(chunk_data)` alongside each chunk and verify it on every read before computing a PoR response. If verification fails, the daemon must report the failure to the microservice immediately (with `audit_result = FAIL` and a specific corruption code in the receipt) rather than computing a wrong hash and letting the microservice deduce it.
(2) **Non-contiguous chunk placement.** Chunks must not be stored in contiguous disk regions. The storage engine must spread chunk placement to reduce the risk that a single surface scratch or spatial media failure corrupts multiple chunks from the same provider simultaneously. This directly reduces within-provider burst failure events.
*Because:* The paper's spatial locality analysis (Section 2.3.2) and the phantom write failure class both identify storage layout as a design choice with reliability consequences.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED`
The four adversarial defence classes (Honest Geppetto, outsourcing, JIT retrieval, false audit responses) remain complete for their intended scope. Physical disk corruption produces audit failures identical to adversarial deletion, but does not require new adversarial defences — the audit system detects it, and the repair system fixes it, regardless of the cause. ADR-014's defences are for intentional misbehaviour; hardware failure is handled by the existing audit+repair loop.
*Because:* The fail-partial model confirms that the audit layer cannot and does not need to distinguish hardware failure from adversarial deletion — both are equivalent from a data availability standpoint.

---

## Disagreements

- **Conventional file system design (ext3, JFS, NTFS):** All ignore write error codes for many block types (DZero for write failures). This is the paper's central finding and its most serious criticism.
*Implication for us:* The provider daemon's write path to local storage must check and handle all write error return codes. A phantom write (success reported but data never stored) caught at write time allows immediate retry or error reporting, rather than waiting for the next audit challenge 24 hours later.
- **RAID as sufficient redundancy:** The paper argues RAID alone does not protect against failures above the disk layer (buggy controllers, firmware errors) and is not available on single-disk systems.
*Implication for us:* Providers in Vyomanaut run on single-disk desktops — RAID is not assumed. Our distributed redundancy (RS(16,56) across 56 providers) provides the cross-machine protection; the per-provider storage engine must provide the within-machine detection the paper describes.

---

## Open Questions

See open-questions.md — questions Q25-1 and Q25-2.