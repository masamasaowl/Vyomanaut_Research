# ADR-023 — Provider-Side Storage Engine: WiscKey Key-Value Separation

**Status:** Accepted
**Topic:** #16 Provider-Side Storage Engine
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 27 (WiscKey, Lu et al., USENIX FAST 2016), Paper 26 (LSM-Tree, O'Neil et al.), Paper 25 (IRON File Systems)

---

## Context

The provider daemon must store and retrieve 256 KB chunks on local desktop and NAS hardware. Three constraints drive the design. First, writes must not amplify I/O so severely that a steady upload stream saturates background bandwidth — the ≤5% CPU and background I/O budget is shared with audit response (ADR-009). Second, audit challenge lookup must complete well within the response deadline of `(chunk_size / declared_upload_speed) × 1.5`; at 256 KB and 5 Mbps declared speed this is ~614 ms. Third, per-chunk content integrity must be verifiable at read time before the PoR response is computed — silent disk corruption must surface as an audit FAIL, not as a wrong hash (IRON, Paper 25).

Paper 26 established that at 256 KB value sizes an LSM-tree that stores keys and values together still has write amplification of 10–14×, and that LSM is inappropriate as the value store itself. Paper 27 (WiscKey) provides the concrete solution: keep only the small chunk index in the LSM; store the actual 256 KB chunk data in a separate append-only value log. This reduces write amplification to approximately 1 at our value size.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Standard LSM (RocksDB with keys and values together) | Mature; no custom code | Write amplification 10–14× at 256 KB; compaction I/O competes with audit reads; LSM too large to stay cached |
| Flat object store (one file per chunk, random disk offset) | Zero write amplification; write-once access pattern is a natural fit | No built-in index; requires an external lookup table anyway; no Bloom filter support; GC is entirely manual |
| **WiscKey-style key-value separation (RocksDB for chunk index, append-only vLog for chunk data)** | Write amplification ≈ 1 at 256 KB; LSM index small and fully cacheable; Bloom filters eliminate disk I/O for absent chunks; sequential vLog appends are HDD-optimal | GC required on chunk deletion; vLog management adds operational code |

## Decision

Adopt WiscKey-style key-value separation as the provider storage engine.

**Component 1 — Chunk Index (RocksDB):**

RocksDB instance storing entries of the form:

```
key:   chunk_id        BYTEA[32]   (SHA256 of chunk content — content address)
value: vlog_offset     uint64      (byte offset into the vLog file)
       chunk_size      uint32      (always 262144 for V2 — lf=256 KB)
```

Entry size: 32 + 8 + 4 = 44 bytes. RocksDB's built-in Bloom filters (10 bits per key, ~1% false-positive rate) eliminate disk reads for absent chunk_ids. An audit challenge for a chunk not assigned to this provider hits only the Bloom filter in memory — no disk I/O, immediate FAIL return path.

Effective write amplification at 256 KB values: `(10 × 44 + 262144) / (44 + 262144) ≈ 1.002`. Negligible.

**Component 2 — Value Log (vLog):**

A single append-only file on the provider's storage device. Each vLog entry:

```
chunk_id        BYTEA[32]           (copy for GC validation — matches RocksDB key)
chunk_size      uint32              (262144 for V2)
chunk_data      BYTEA[262144]       (raw 256 KB chunk, AONT-transformed and RS-coded by the client)
content_hash    BYTEA[32]           (SHA256(chunk_data) — verified on every read before PoR response)
```

Entry size: 32 + 4 + 262144 + 32 = 262212 bytes ≈ 256.2 KB.

The `content_hash` is verified on every vLog read. If `SHA256(chunk_data) ≠ content_hash`, the daemon reports `audit_result = FAIL` to the microservice immediately and queues the chunk for repair reporting. Silent disk corruption surfaces as an audit failure rather than a wrong PoR response hash (IRON Paper 25 requirement).

**Audit challenge lookup path:**

1. Provider daemon receives `(chunk_id, challenge_nonce)` from the microservice.
2. Query RocksDB for `chunk_id`.
   - Bloom filter absent → return `audit_result = FAIL` with no disk I/O.
   - Found → retrieve `(vlog_offset, chunk_size)` (typically from RocksDB block cache — no disk I/O).
3. Read 262212 bytes from vLog at `vlog_offset`. One random disk read: ~1 ms SSD, ~12–15 ms HDD.
4. Verify `SHA256(chunk_data) == content_hash`. Fail immediately if mismatch.
5. Compute `response_hash = SHA256(chunk_data || challenge_nonce)`.
6. Return signed audit receipt (ADR-017).

Total disk I/Os on cache hit: 1. Both SSD and HDD latencies are well within the 614 ms audit deadline.

**Write path (chunk store at upload time):**

1. Append vLog entry to end of vLog file. Sequential write; HDD-optimal.
2. Compute `content_hash = SHA256(chunk_data)` and embed in the vLog entry.
3. `fsync()` the vLog before proceeding.
4. Insert `(chunk_id, vlog_offset, chunk_size)` into RocksDB.
5. Signal success to the microservice.

The `fsync()` before the RocksDB insert ensures that if the daemon crashes between steps 3 and 4, the vLog entry is durable and will be recovered on restart by the tail scan described below.

**Crash recovery:**

On daemon restart, retrieve `<"vlog_head", head_offset>` from RocksDB. Scan the vLog from `head_offset` to the end of the file. For each vLog entry found, re-insert the RocksDB record if the `chunk_id` is absent. This repairs any index entries lost between the last RocksDB flush and the crash. The maximum scan length is one RocksDB memtable flush interval worth of entries — typically a few hundred chunks.

**Garbage collection:**

Triggered only on chunk deletion events: provider exit (ADR-007 announced departure), file owner deletion request, or repair reassignment by the coordination microservice. GC process:

1. Read a chunk of vLog from the tail (e.g., 4 MB = ~16 entries).
2. For each entry, check RocksDB for the `chunk_id`. If absent from RocksDB, the chunk is invalid.
3. Append valid entries to the vLog head.
4. Update `<"vlog_tail", tail_offset>` in RocksDB synchronously.
5. Use `fallocate(FALLOC_FL_PUNCH_HOLE)` to release the freed vLog region to the file system.

For a cold storage write-once workload, GC is infrequent. A provider holding 50 GB at 256 KB/chunk stores ~200,000 chunks; deletions occur only on provider exit or explicit file owner action. GC overhead is negligible in normal operation.

**Physical layout and placement:**

The vLog is a single contiguous append-only file. Chunks from different files are interleaved by upload time, not grouped by file. A provider holds one RS shard from each of many different files — never all 56 shards of any one file. This means that a spatial disk failure (surface scratch across a contiguous vLog region) corrupts shards from many different files, each of which retains 55 surviving shards across other providers — well above the k=16 reconstruction threshold. The 20% ASN cap (ADR-014) bounds the network-level impact of any one provider's full disk failure. Sparse vLog placement to further reduce within-provider burst correlation is deferred to V3 pending empirical measurement of desktop disk failure patterns (Q27-2).

**RocksDB configuration for background budget:**

```
max_background_compactions = 1
max_background_flushes = 1
low_priority_background_threads = 1
rate_limiter = 10 MB/s   (background compaction I/O cap — tune empirically)
```

These starting values keep RocksDB compaction within the ≤5% CPU budget (ADR-009). Compaction I/O is minimal because the LSM stores only 44-byte index entries, not 256 KB values.

## Consequences

**Positive:**
- Write amplification ≈ 1 at 256 KB: a provider storing 50 GB of chunks consumes ~50 GB of sequential write bandwidth, not 500–700 GB as with standard LevelDB
- Sequential vLog appends are HDD-optimal — most Indian home desktop providers at V2 launch will have HDDs
- RocksDB Bloom filters eliminate all disk I/O for audit challenges on chunks not assigned to this provider
- Audit lookup requires exactly one random disk read once the RocksDB block cache is warm
- `content_hash` per vLog entry detects silent disk corruption before the PoR response is computed, satisfying the IRON end-to-end checksum requirement

**Negative / trade-offs:**
- vLog GC management adds operational code that a flat object store would not require
- The vLog is a single large file; file systems with poor large-file performance (FAT32, some NTFS configurations) must be excluded from provider hardware requirements
- Crash recovery requires scanning the vLog tail — worst case a few hundred chunks (~50 MB) if the daemon died immediately after a vLog fsync but before RocksDB flush

**Open constraints:**
- RocksDB rate limiter threshold (10 MB/s starting value) must be calibrated against actual provider hardware at V2 launch (Q27-1)
- Sparse vLog placement to reduce within-provider spatial failure correlation is a V3 enhancement pending empirical failure data (Q27-2)

## References

- [Paper 27 — WiscKey](../research/paper-27-wisckey.md): key-value separation architecture; write amplification at 256 KB; audit lookup latency; GC design; crash recovery
- [Paper 26 — LSM-Tree](../research/paper-26-lsm-tree.md): theoretical basis for key-value separation; batching parameter M; Bloom filter requirement
- [Paper 25 — IRON File Systems](../research/paper-25-iron-file-systems.md): content_hash per chunk requirement; non-contiguous placement constraint
- [ADR-002](ADR-002-proof-of-storage.md): PoR challenge response format; audit_result field
- [ADR-009](ADR-009-background-execution.md): ≤5% CPU background budget
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap bounds within-provider burst failure impact at network level
- [ADR-017](ADR-017-audit-receipt-schema.md): audit receipt schema; response_latency_ms field
