# ADR-046 — Provider Storage Engine on Windows: BadgerDB, Not RocksDB/CGo

**Status:** Proposed
**Topic:** #16 Provider-Side Storage Engine, #22 Desktop Application Shell Architecture (Windows build)
**Supersedes:** [ADR-023](./ADR-023-provider-storage-engine.md) — **Windows target only.** ADR-023's WiscKey design, audit-latency reasoning, and Linux/macOS implementation are unchanged; only the Windows build's engine binding is superseded here.
**Superseded by:** —
**Research source:** Paper 49 (BadgerDB); `ci.yml` (existing CI — no Windows or arm64-cross RocksDB coverage); facebook/rocksdb #1920, #3984 (upstream Windows/MinGW support status)

---

## Context

ADR-023 (Accepted) specifies the provider storage engine as WiscKey key-value separation: RocksDB (via `linxGnu/grocksdb`, a CGo binding) for the chunk index, plus hand-written Go code for the value log, single-writer goroutine, crash recovery, and garbage collection. This is implemented and CI-tested — but only on Linux (via a pinned Docker image with RocksDB pre-built) and macOS (via a native `macos-15` runner installing RocksDB through Homebrew). The CI pipeline's own comments already state why: `internal/storage` needs CGo plus a RocksDB build for the target platform, and cross-compiling that combination is not attempted because there is no RocksDB build for the target in the available images. There is no Windows equivalent of the macOS native-runner workaround, and none has been attempted.

Vyomanaut's desktop applications ship on Windows first (`ux-decisions.md` §7; ADR-038, ADR-041). The provider daemon (`cmd/provider`, M13) depends on `internal/storage` directly — this is not a peripheral package. Left unaddressed, the first attempt to build a Windows provider daemon is also the first attempt to build `grocksdb` on Windows at all, on a milestone's critical path.

Two upstream facts (Paper 49) make this a real risk, not a routine porting task:

- RocksDB's own CMake build supports MSVC on Windows and explicitly does not support MinGW (facebook/rocksdb #1920: *"the cmake-based build system doesn't support anything other than MSVC on windows"*).
- Go's CGo toolchain on Windows conventionally requires a MinGW-family C compiler, not MSVC, as `$CC`. A prior community attempt to build RocksDB with MinGW-w64 hit a runtime assertion failure tied to Windows-specific thread-local-storage handling (facebook/rocksdb #3984) — meaning even a successful compile is not confirmed to produce a correct binary.

Closing this gap by hardening a native Windows RocksDB/CGo pipeline (MSVC + vcpkg, then bridging MSVC-built objects into a Go CGo build) is possible in principle but is unproven, effort-intensive, and has no existing precedent in this codebase to build from. Paper 49 identifies a lower-risk alternative already available: BadgerDB, a mature, pure-Go implementation of the same WiscKey design ADR-023 is built on, with zero CGo dependencies in its current major version.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Invest in a native Windows RocksDB/CGo pipeline (MSVC + vcpkg, validated on a `windows-latest` CI runner) | Keeps one storage-engine implementation across all platforms | Unproven — no existing precedent in this codebase or the upstream RocksDB Windows docs for bridging an MSVC-built lib into a Go CGo binary; the one documented MinGW attempt (facebook/rocksdb #3984) hit a runtime bug, not just a build failure; open-ended effort on a milestone critical path |
| Cross-compile `grocksdb` for Windows from the existing Linux/macOS CI images via a MinGW cross-toolchain | Avoids needing a Windows build machine at all | Same MinGW-vs-MSVC-official-support gap applies; still unproven; the existing CI's own comments already treat RocksDB cross-compilation (for arm64) as infeasible for exactly this reason |
| **BadgerDB (pure Go, CGo-free) as the Windows-only provider storage engine** | Implements the same WiscKey design already cited by ADR-023 (Paper 27); zero CGo dependency in current major version (v4); Bloom filter and block cache carry over, preserving the audit-latency argument; proven at production scale (Dgraph, Jaeger Tracing) | A second engine implementation to build, test, and keep behaviorally consistent with the Linux/macOS RocksDB path; on-disk layout differs from ADR-023's byte-exact RocksDB/vLog format, so the HDD compaction benchmark must be re-run against Badger specifically, not assumed to transfer |
| BadgerDB on every platform, retiring RocksDB entirely | One engine implementation everywhere; removes the CGo/CI-fragility class of problem network-wide, not just on Windows | Not evaluated here — Linux/macOS already has a working, CI-proven RocksDB path; replacing it is a larger decision requiring its own performance validation on real provider hardware (tracked as Q49-1), not something to decide as a side effect of unblocking Windows |

## Decision

**Windows builds of `internal/storage` use BadgerDB v4 in place of `grocksdb`/RocksDB. Linux and macOS builds are unchanged — they continue to use ADR-023's RocksDB/vLog implementation, which already has a working, CI-validated build path on both.**

### 1. Package structure

`internal/storage` exposes one Go interface (the same `AppendChunk` / `LookupChunk` / `DeleteChunk` / `RunGC` / `RecoverFromCrash` / `Close` surface already specified by ADR-023 and `build.md` Session 5.1.x) with two backends selected by Go build tags:

```
internal/storage/engine_rocksdb.go   // go:build linux || darwin — ADR-023's implementation, unchanged
internal/storage/engine_badger.go    // go:build windows — this ADR
```

No caller-visible change: `cmd/provider` and every other package that consumes `internal/storage` calls the same interface regardless of which file was compiled in.

### 2. Badger configuration

The Windows backend configures Badger to reproduce the properties ADR-023's audit-latency budget depends on, not Badger's general-purpose defaults:

```go
opts := badger.DefaultOptions(dbPath).
    WithValueThreshold(1024).           // route every 256 KB chunk to the value log — never inline in the LSM
    WithBloomFalsePositive(0.01).       // matches ADR-023's 10-bits/key ≈ 1% target on the RocksDB path
    WithBlockCacheSize(64 << 20).       // 64 MB — matches the RocksDB path's block cache size
    WithCompression(options.None)       // chunk data is already encrypted + erasure-coded (ADR-019/ADR-022) —
                                         // ciphertext does not compress; spending CPU trying violates the
                                         // ≤5% background CPU budget (ADR-009) for no benefit
```

`content_hash` verification (SHA-256 of `chunk_data`, checked before every audit response — ADR-023's IRON-derived integrity requirement) is unchanged: it is application logic sitting on top of `Get`/`Set`, not something either engine provides natively, so it is identical on both backends.

### 3. What Badger removes, not just replaces

Adopting Badger on Windows does not just swap one dependency for another — it removes code. ADR-023's single-writer goroutine (working around `O_APPEND` non-atomicity at 262,212-byte entries), its manual crash-recovery tail scan, and its manual GC pass are all things Badger's engine already does internally. The Windows backend does not need a `engine_badger.go` reimplementation of that machinery; it needs a thin adapter from Vyomanaut's storage interface to Badger's `Txn`-based API.

### 4. Build and CI implications

Because Badger has no CGo dependency, `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./internal/storage/...` succeeds from the *existing* Linux CI container with no new toolchain — this is a plain Go cross-compile, not a cross-compile of a C++ dependency. This closes the build-verification gap immediately, without waiting on a native Windows runner. A native `windows-latest` runner (mirroring the existing native `macos-15` job) is still needed to actually *run* `internal/storage`'s tests and the HDD-equivalent compaction benchmark — cross-compiling proves the code compiles for Windows, not that it behaves correctly there.

## Consequences

**Positive:** removes the single largest unaddressed build risk on the Windows critical path before M13 begins; the Windows provider build no longer depends on an unproven MSVC/MinGW bridging effort; less code to own on the Windows path (no hand-rolled vLog/GC/crash-recovery); build-verification is available immediately via cross-compilation, with no new CI toolchain required.

**Negative / trade-offs:** two storage-engine implementations to test and keep behaviorally consistent (same interface, different internals, different failure modes); the HDD compaction benchmark (ADR-023 § "HDD-specific compaction benchmark") must be re-run against Badger's own compaction/compression knobs (`NumCompactors`, memtable size) rather than assumed to inherit RocksDB's tuned values; a provider's on-disk chunk store is not portable between the Windows and Linux/macOS formats (not a practical concern — a running provider's OS does not change under it).

**Open constraints:**

- **Q49-1** (Paper 49): whether Badger's throughput advantage over general-purpose RocksDB at 256 KB values justifies retiring RocksDB on Linux/macOS too, unifying on one engine. Not blocking — explicitly deferred to post-launch measurement against real provider hardware, since the existing RocksDB path is not broken.
- The HDD compaction pass/fail criterion (p99 audit latency ≤ 200 ms under active compaction, 7200 RPM HDD) must be re-validated against Badger's own tuning surface before the Windows provider daemon ships.
- A native `windows-latest` CI runner exercising `internal/storage`'s test suite (not just its build) should be added alongside the existing native `macos-15` job.
- `ValueThreshold`, `BloomFalsePositive`, and block cache size above are starting values, consistent with ADR-023's own "tune empirically" precedent for its RocksDB rate limiter — not final.

## References

- Paper 49 — BadgerDB documentation, source, and production track record
- Paper 27 — WiscKey (already cited by ADR-023; Badger's own stated design basis)
- [ADR-023](./ADR-023-provider-storage-engine.md) — the WiscKey decision and Linux/macOS implementation this ADR leaves unchanged
- [ADR-009](./ADR-009-background-execution.md) — ≤5% CPU background budget (motivates disabling compression on already-encrypted chunk data)
- facebook/rocksdb #1920 — Windows CMake build supports MSVC only, not MinGW
- facebook/rocksdb #3984 — MinGW-built RocksDB runtime assertion failure (Windows thread-local storage)
