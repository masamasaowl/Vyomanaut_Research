# ADR-046 — Addendum A: Full-Adapter Resolution, Not Index-Only Substitution

**Applies to:** [ADR-046 — Provider Storage Engine on Windows: BadgerDB, Not RocksDB/CGo](./ADR-046-badgerdb-windows-storage-engine.md)
**Status of parent ADR:** Accepted — **unchanged in intent.** ADR-046 §3's stated architecture (Badger's own crash-recovery/GC machinery replaces the hand-rolled vLog machinery on Windows) is exactly what this addendum implements. What changes is a factual correction to ADR-046 §1's description of the *existing* file layout, and the concrete file-level plan that follows from that correction.
**Added research source:** none — this addendum records an implementation-planning correction and its resolution, not new external research.
**Findings addressed:** none numbered in the existing F-XX corpus; recorded here as build.md Session 16.0.1's own build-blocker/Design-Council record.

> **Insert location:** after ADR-046 §1 ("one Go interface... with two backends selected by build tags"), as a correction to that paragraph's description of `engine_rocksdb.go`'s contents.

---

## Why this addendum exists

ADR-046 §1 states that `internal/storage` would expose one `ChunkStore` interface with two backends, and describes `engine_rocksdb.go` (the build-tag-renamed `index.go`) as already containing the six-method `ChunkStore` surface (`AppendChunk`, `LookupChunk`, `DeleteChunk`, `RunGC`, `RecoverFromCrash`, `Close`) that `engine_badger.go` would need to match.

That description was wrong about the *existing* code, not about the target architecture. Reading the live M5 implementation (Session 16.0.1, before writing any new code) found:

- `index.go` (→ `engine_rocksdb.go`) defines `rocksDBIndex`, exposing only lowercase, unexported low-level methods: `put`, `get`, `del`, `allChunkIDs`, `close`.
- The six `ChunkStore` methods are implemented in **`vlog.go`**, on a separate type `wiskeyStore`, which owns `index *rocksDBIndex` (a **concrete pointer**, not an interface) plus the raw `*os.File` value log, and implements `AppendChunk`/`LookupChunk`/etc. by combining index operations with direct file I/O.

`vlog.go` was not in Session 16.0.1's `FILES:` scope. Session 16.0.1's own `VERIFY:` block (`CHUNKSTORE_INTERFACE_SURFACE_IDENTICAL_ON_BOTH_ENGINES`) greps `engine_rocksdb.go` for the six method names and expects `>= 6` — a check that cannot pass against the real file layout no matter how the session is executed, since those methods have never lived there.

This was raised as a build blocker before any implementation code was written, then resolved by a five-seat Design Council (`/design-council`, session record: "Windows Storage Engine: Full-Adapter vs. Index-Only Badger Integration").

## The two resolutions considered

**Resolution 1 (adopted) — Full adapter.** `engine_badger.go` implements all six `ChunkStore` methods directly against `badger.DB`, storing full 256 KB chunk values through Badger's own value log, GC, and crash recovery. `store.go`'s `NewChunkStore` becomes a thin, build-tag-selected dispatcher (`newEngineStore`) to one of two independent constructors — one per engine file. `vlog.go` stays **byte-for-byte unchanged**; only the call site that constructs `wiskeyStore` moves, from `store.go` into `engine_rocksdb.go`.

**Resolution 2 (rejected) — Index-only substitution.** Badger stands in only for `rocksDBIndex`'s low-level role; `vlog.go`'s hand-rolled single-writer/tail-scan/GC file logic keeps running on Windows too.

## Why the Council rejected Resolution 2

Five independently-argued seats converged on Resolution 1 from different angles (full council record: build.md Session 16.0.1):

- **Audit-integrity risk (Adversary seat):** Resolution 2 would require Windows to reuse `vlog.go`'s SHA-256-against-`wantChunkID` verification path unmodified, which is fine — but forces `wiskeyStore.index` from a concrete `*rocksDBIndex` to an interface type, a structural edit to a file the session explicitly promises stays unchanged.
- **NFR-024 durability contract (Systems Theorist seat):** Resolution 2 implicitly assumes Badger's LSM provides the same WAL-durability guarantee RocksDB does *for a component that no longer owns the data it's indexing* — an unstated, unverified assumption with no corresponding DM section.
- **ADR-046's own benchmark requirement (Scale Advocate seat):** ADR-046 §4 requires an HDD compaction benchmark re-run against Badger's real compaction/compression knobs before shipping. Under Resolution 2, Badger only ever stores 12-byte offset/size pairs — that benchmark could never be run against the real 256 KB chunk workload it's meant to validate.
- **Provenance (Outsider seat):** Resolution 2's only actual argument was "fewer file edits" — an artifact of the session text's `FILES:` scoping, not a reason anyone would choose it on engineering merits, given ADR-046 §3 already specified the full-adapter approach in plain text.
- **Regression risk (Implementer seat):** Resolution 2 edits the *proven* Linux/macOS path (`vlog.go`) to accommodate the *new* platform; Resolution 1 leaves it untouched and confines all new code to a new file plus a thin dispatcher.

## What Resolution 1 required, corrected during implementation

Executing Resolution 1 surfaced one further correction the Council's own text didn't anticipate: **`vlog.go` needed a `//go:build linux || darwin` tag it didn't have.** Once `engine_rocksdb.go` carried that tag, `wiskeyStore`'s `index *rocksDBIndex` field type stopped existing under a Windows build — `vlog.go` itself has no build constraint, so Go still tried to compile it (and fail) on every platform. This is a one-line, zero-logic-change fix, caught by an actual `GOOS=windows` cross-compile, not by inspection. `vlog.go`'s diff for this addendum is exactly that one line.

## Implementation record

- `internal/storage/engine_rocksdb.go` (renamed from `index.go`): `//go:build linux || darwin` added; `newEngineStore` (the Linux/macOS constructor, relocated verbatim from `store.go`'s old `NewChunkStore` body) and a compile-time `var _ ChunkStore = (*wiskeyStore)(nil)` assertion appended. No RocksDB-specific logic changed.
- `internal/storage/engine_badger.go` (new): `badgerStore`, implementing all six `ChunkStore` methods directly against `badger.DB`, configured per ADR-046 §2 (`WithValueThreshold(1024)`, `WithBloomFalsePositive(0.01)`, `WithBlockCacheSize(64<<20)`, `WithCompression(options.None)`).
  - `AppendChunk`'s returned `vlogOffset` is always `0` on this engine — Badger does not expose internal value-log byte offsets through its public API, and no caller in `cmd/provider` reads this value beyond passing along the error (confirmed by grep).
  - `AppendChunk` failures map to the shared `ErrVLogFsync` sentinel; `ErrRocksDBInsert` never applies on this path, since a single `badger.Txn` commit is atomic — there is no window where chunk data is durable but its index entry is missing, unlike the two-phase RocksDB write.
  - `RecoverFromCrash` is a documented no-op: `badger.Open` (called from `newEngineStore`) already performs Badger's own crash recovery synchronously before this store is ever constructed. This satisfies the shared `ChunkStore` precondition ("no `AppendChunk` since open, no writer goroutine running") trivially rather than by coincidence — `cmd/provider/main.go`'s startup sequence guarantees that ordering regardless of which engine is compiled in.
  - `RunGC` loops `db.RunValueLogGC(0.5)` until `badger.ErrNoRewrite`, replacing ADR-023's hand-rolled compact-to-tmp-then-rename pass entirely on this platform.
- `internal/storage/engine_badger_test.go` (new): `TestBadgerAppendLookupDeleteRoundTrip`, `TestBadgerContentHashVerifiedBeforeAuditResponse`, `TestBadgerAndRocksDBSatisfySameChunkStoreInterface` — the last of these is a compile-time-only check on this platform, since `engine_rocksdb.go`/`vlog.go` are never compiled into a Windows test binary; the RocksDB-side half of the same interface assertion lives in `engine_rocksdb.go` itself.
- `internal/storage/store.go`: `NewChunkStore` reduced to `os.MkdirAll` plus a call to the build-tag-selected `newEngineStore` — no OS-specific code of its own.
- `.github/workflows/ci.yml`: new `storage-windows-coverage` job, native `windows-latest`, `continue-on-error: true` pending its first real run (same "PENDING" pattern already used for other new checks in this file).
- `go.mod`: `github.com/dgraph-io/badger/v4 v4.5.1` added as a direct dependency, plus its real transitive indirect requires.

## Verification performed

- `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build`/`go vet` on `internal/storage`: clean, in an isolated module mirroring the package's full dependency closure.
- The three `engine_badger_test.go` tests: run for real (Badger is pure Go — no cross-compile restriction on execution), by temporarily relaxing the build tag to `linux` in an isolated verification copy only, never in the committed files. All three passed.
- Native Linux `go build`/`go test` for `internal/storage` (the RocksDB path): **not independently verified in the sandbox this work was done in** — that sandbox's `apt`-installed RocksDB (8.9.1) doesn't match the pinned `grocksdb v1.10.8` binding's expected RocksDB 10.x API surface, and the project's real CI image (`rocksdb-10.10.1-pgclient1`) wasn't reachable from that sandbox's network allowlist. Every symbol the new `newEngineStore` (in `engine_rocksdb.go`) touches was traced back to code that already compiled successfully before this session (`openRocksDBIndex`, and a `wiskeyStore{...}` literal identical field-for-field to the one previously inline in `store.go`) — nothing in the diff exercises new RocksDB-specific surface. The real `windows-latest` and native Linux/macOS CI jobs are the actual gate for this; this reasoning is not a substitute once the change lands in a PR.

## What this does not close

- The HDD compaction benchmark ADR-046 §4 requires (Badger's real compaction/compression knobs against the 256 KB chunk workload, on rotational storage) has not been run. This addendum's Resolution 1 is what makes that benchmark *possible* to run meaningfully; it does not run it.
- `storage-windows-coverage`'s `continue-on-error: true` should be revisited once a first real `windows-latest` run confirms there's nothing platform-specific (file locking, path separators) the cross-compile check alone can't catch.
