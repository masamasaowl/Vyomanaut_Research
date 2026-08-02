# Paper 49 — BadgerDB: Documentation, Source, and Production Track Record

**Authors:** Dgraph Labs / Hypermode (dgraph-io/badger maintainers); Manish R. Jain (Badger's original author)
**Venue / Year:** Official project documentation and repository (badger.dgraph.io, github.com/dgraph-io/badger) | v4 current, retrieved 2026
**Citations:** not applicable — documentation and source, not an academic paper. Its own design is a direct implementation of Paper 27 (WiscKey), already cited by ADR-023.
**Topics:** #16 Provider-Side Storage Engine, #22 Desktop Application Shell Architecture (Windows build)
**ADRs produced:** ADR-046 (BadgerDB as the Windows provider storage engine); addendum to ADR-023 (Windows scope only)

---

## Problem Solved

ADR-023 (Accepted) builds the provider storage engine as a hand-rolled WiscKey implementation: RocksDB for the chunk index, plus custom Go code for the value log, single-writer goroutine, crash recovery, and garbage collection. RocksDB is consumed through `linxGnu/grocksdb`, a CGo binding. The CI pipeline already documents, in its own comments, that this binding has no working Windows path and no working arm64 cross-compile path — a native macOS runner was added specifically to dodge cross-compiling it, and Windows has no equivalent. This review asks whether a pure-Go implementation of the same WiscKey design can remove the CGo dependency entirely, rather than asking the Windows build to solve a problem RocksDB's own upstream has not solved for MinGW toolchains.

---

## Key Findings

**Badger is not RocksDB-with-a-Go-face — it is an independent, from-scratch implementation of the same paper ADR-023 already cites.** Its own documentation states its design is based on the WiscKey paper (Paper 27), for the same reason ADR-023 gives: separating large values from the LSM index cuts write amplification versus a standard LSM. This means adopting Badger is not introducing a new, unvalidated architecture — it is swapping the implementation of an architecture already accepted, for one that implements it natively rather than as a RocksDB-plus-hand-rolled-vLog composite.

**Badger v4 has zero CGo dependencies, confirmed in its current `go.mod`.** Earlier versions optionally used `DataDog/zstd` (a CGo binding) for ZSTD compression; this was replaced by `klauspost/compress`, a pure-Go ZSTD implementation, merged and released years ago. The current v4 dependency tree (`cespare/xxhash`, `dgraph-io/ristretto`, `klauspost/compress`, `google/flatbuffers`, protobuf, OpenTelemetry) contains no C bindings anywhere. `CGO_ENABLED=0 go build` works unconditionally — there is no compression feature that requires re-enabling CGo, unlike in v2/v3.

**Badger already implements every piece of custom machinery ADR-023 had to hand-build.** ADR-023's decision text specifies: a single-writer goroutine serializing vLog appends (`O_APPEND` is not atomic at 262,212-byte entries), a crash-recovery tail scan against a stored head pointer, and a manual GC pass reading vLog tail entries and checking index presence. Badger's engine performs all three internally — a configurable value-log threshold (`Options.ValueThreshold`, set low so all 256 KB chunks route to the value log rather than the LSM), automatic crash recovery on open, and `RunValueLogGC()` for reclaiming deleted-entry space. Adopting Badger for a platform does not just remove `grocksdb` — it removes the custom vLog/GC/crash-recovery code that currently exists to work around RocksDB not doing this natively, since RocksDB (a general LSM, not a WiscKey engine) has no equivalent concept of its own.

**Bloom filters and block cache — the two properties ADR-023's audit-latency math depends on — carry over.** ADR-023's fast-fail path for an unassigned `chunk_id` (no disk I/O, Bloom-filter negative) and its warm-cache index lookup (typically zero disk I/O) both depend on RocksDB features that Badger also implements: a configurable table-level Bloom filter (`Options.BloomFalsePositive`, default 1% — the same false-positive rate ADR-023 specifies for RocksDB's 10-bits-per-key filter) and an LSM block cache (`Options.BlockCacheSize`). The audit-deadline math in ADR-023 (≤614 ms at 256 KB / 5 Mbps) does not depend on anything RocksDB-specific; it depends on "index lookup is Bloom-filtered and cached, value read is one disk seek" — a property Badger provides by the same mechanism.

**Badger is proven at a scale relevant to Vyomanaut's target, not a toy.** It backs Dgraph (a production graph database), is used by Jaeger Tracing and other projects, and its own documentation states it is "being used to serve data sets worth hundreds of terabytes," with a nightly 8-hour Jepsen-style consistency test running under the race detector. This is materially more independent production validation than a from-scratch storage engine would have.

**The write-amplification argument against general-purpose RocksDB at large value sizes is not specific to Vyomanaut's implementation choice — it is an active, current research finding.** A 2026 paper on large-value KV storage (arXiv:2602.01873) benchmarks RocksDB and RocksDB's own large-value extension (BlobDB) directly, finding a purpose-built large-value store achieves 8.4× RocksDB's write throughput and 1.7–15.6× its read throughput across get/exists/scan operations, because RocksDB's general-purpose LSM still pays compaction and traversal costs a value-log-native design avoids. This corroborates ADR-023's own reasoning (Papers 25–27, 32, 34) about why raw RocksDB was never the intended end state — WiscKey separation was the fix, and Badger applies it without the CGo cost RocksDB reintroduces at the binding layer.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Badger's built-in value log, GC, and crash recovery | ADR-023's hand-rolled equivalents on top of RocksDB | Removes code Vyomanaut currently owns and must test itself, at the cost of losing fine-grained control over the exact on-disk layout ADR-023 specifies byte-for-byte (44-byte index entry, 262,212-byte vLog entry) — Badger's own entry framing differs and is not identical to the hand-specified format |
| Pure-Go, CGo-free build (v4) | `grocksdb`'s CGo binding | Removes the entire class of toolchain risk this review exists to close, at the cost of depending on Badger's own compaction/GC tuning rather than hand-tuned RocksDB options already tuned in ADR-023 (`rate_limiter`, `max_background_compactions`) |
| Adopting an independent WiscKey implementation | Continuing to invest engineering time in a native Windows RocksDB/CGo build (MSVC + vcpkg + toolchain validation) | Avoids sinking effort into a build problem with no confirmed solution yet, at the cost of a second storage-engine implementation to validate if this is scoped to Windows only rather than adopted everywhere |

---

## Breaks in Our Case

- **Badger is a general-purpose embedded KV store used across many workloads (graphs, tracing spans, arbitrary application data)**
  ≠ **Vyomanaut's provider daemon has exactly one access pattern: fixed 256 KB values, content-addressed by SHA-256, write-once, read-rarely, deleted only on provider exit or repair reassignment**
  → Vyomanaut doesn't need Badger's transaction/MVCC/SSI machinery at all — the daemon needs `Set`/`Get`/`Delete` and nothing else. This isn't a mismatch so much as unused surface area: the features ADR-023 needs (Bloom filter, block cache, value-log separation) are exactly Badger's core, and the features it doesn't need (transactions, versioning) are inert rather than harmful.

- **ADR-023 specifies exact on-disk byte layouts (44-byte RocksDB entries, `content_hash` embedded per vLog entry, manual `fallocate(FALLOC_FL_PUNCH_HOLE)` GC)**
  ≠ **Badger owns its own on-disk format and does not expose a hook to replicate that exact layout**
  → This is a real, not cosmetic, difference: adopting Badger means trusting its internal format and GC behavior rather than the byte-exact design ADR-023 walks through. The content-integrity property (`SHA256(chunk_data)` checked before every audit response) is still directly implementable as application-level logic on top of Badger's `Get`/`Set` — it does not depend on owning the physical layout — but this should be stated as a deliberate trade, not silently assumed identical.

- **ADR-023's HDD-specific compaction benchmark (§ "HDD-specific compaction benchmark") was written and tuned against RocksDB's specific rate-limiter and compaction-thread options**
  ≠ **Badger has its own compaction and compression tuning surface (`NumCompactors`, `Options.Compression`, `MemTableSize`, `ValueLogFileSize`) with different defaults and different knobs**
  → The pass/fail criterion (p99 audit latency ≤ 200 ms under active compaction on 7200 RPM HDD) is engine-agnostic and should be re-run against Badger directly rather than assumed to transfer from the RocksDB-tuned values.

---

## Decisions Influenced

- **ADR-046 (BadgerDB as the Windows provider storage engine)** `NEW`
  Badger replaces `grocksdb`/RocksDB for the Windows build of `internal/storage`, closing the CGo/toolchain risk documented against ADR-023.
  *Because:* it implements the same paper ADR-023 is already grounded in, has no CGo dependency to fail on Windows, and carries the two properties (Bloom filter, block cache) the audit-latency budget actually depends on.

- **ADR-023 (Provider-Side Storage Engine)** `ADDENDUM — WINDOWS SCOPE ONLY`
  RocksDB via `grocksdb` remains the engine on Linux/macOS, where it already has proven, CI-validated build paths. The Windows target uses Badger instead. This is a platform-scoped fork, not a reversal — see the addendum to ADR-023 and ADR-046 for the full reasoning, including the open question (Q49-1) of whether this should eventually become the single engine on every platform.

---

## Disagreements

- **A case can be made that maintaining two storage-engine implementations (RocksDB+vLog on Linux/macOS, Badger on Windows) is worse than the CGo risk it solves — two GC/crash-recovery models to test, two sets of edge-case bugs, two behaviors to keep in parity.**
  *Implication for us:* this review does not resolve that trade-off. It establishes that Badger is a safe, well-precedented way to unblock the Windows build specifically. Whether the long-term right answer is "Badger everywhere" (matching CockroachDB's own move away from RocksDB, for closely related reasons) is a separate, larger decision — tracked as Q49-1, not decided here.

---

## Open Questions

See `requirements.md` §10.1 — question Q49-1: does Badger's write/read throughput advantage over general-purpose RocksDB at 256 KB values (suggested by Badger's own published comparisons and by the 2026 large-value KV benchmark above) hold on Vyomanaut's actual Linux/macOS provider hardware and workload, to a degree that justifies retiring the RocksDB/vLog path everywhere rather than only on Windows? Not blocking — the CI-proven RocksDB path on Linux/macOS is not broken, so this is a performance/maintenance-burden question to answer empirically post-launch, not a pre-launch blocker.

*(Note: this and every other Phase 7/8 paper cite `requirements.md` §10.1 directly, not `open-questions.md` — the latter is referenced by nearly every earlier paper in this repo but does not exist as a file. Flagged, not fixed retroactively, in this pass; see the note attached to Paper 50.)*
