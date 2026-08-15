# ADR-074 — Repair Pipeline Live-Verification Findings (Session 16.1.1, continued)

**Status:** Accepted
**Track:** DEMO
**Topic:** #4 Repair Protocol, #11 Upload Assignment Service (file-registration half only)
**Supersedes:** — *(continues the live-verification effort ADR-070 and ADR-073 opened; does not reopen either)*
**Superseded by:** —
**Research source:** none — this ADR records implementation-gap findings from live end-to-end execution, not new external research.

---

## Context

ADR-073 unblocked real, live uploads (client-submitted, per-segment content-hash `chunk_id`) but explicitly scoped itself to assignment — it named `internal/repair/executor.go` and `internal/vettingchunk/generator.go` as **out of scope**, on the reasoning that both already mint capability tokens against a `chunk_id` that is a real content hash at mint time. That reasoning is correct as far as it goes; it does not mean the repair pipeline had ever actually been run live. It hadn't. Session 16.1.1's continuation picked up exactly where ADR-073 left off: running the full demo timeline — file registration, departure detection, repair — against real infrastructure (real Postgres, real `cmd/microservice`, real `cmd/provider` processes) for the first time.

Four further findings surfaced, in the order hit — the same shape ADR-070 already named for the onboarding lifecycle: a component correctly implementing its own documented contract, defeated by a neighbouring component that disagreed with it at the boundary, invisible to either side's own unit tests because neither exercises the other's code.

## Findings

### F-16-1 — File registration's `owner_sig` verification used JSON-marshaled input, not the fixed-layout signing convention

`internal/api/file.go`'s original (Session 11.7.2) `owner_sig` verification followed a hand-built canonical-JSON-shaped byte string, verified via a bare `ed25519.Verify` call — by its own header comment, deliberately distinct from `provider.go`'s `crypto.SignBytes`/`VerifyBytes` hash-then-sign convention. This is incompatible with `internal/crypto`'s own package doc, which states as a CRITICAL rule that JSON serialisation must never be used for signing inputs (field ordering is not guaranteed across Go versions) and that all signing inputs must be a fixed-layout byte sequence. `internal/client/upload/pointer.go`'s client-side signing had already been corrected to a fixed-layout scheme (finding A-6); `file.go` was never updated to match, so every real `POST /api/v1/file/register` call — the final step of upload — failed `owner_sig` verification deterministically.
**Fix:** `ownerSigSigningInput` rebuilt as a fixed-layout byte sequence matching `pointer.go`'s `computeOwnerSig` exactly; verification now goes through `localcrypto.VerifyBytes` (IC §3.2's actual composition).
**Confirmed live:** file registration — the final upload step — now succeeds.

### F-16-3 — Repair-download Frame 1 was missing `request_ts_ms` on the wire

`internal/repair/executor.go`'s `downloadOneShard` computed `repair_auth_sig` over `chunk_id ‖ request_ts_ms ‖ microservicePeerID`, but only ever transmitted `chunk_id ‖ repair_auth_sig` (96 bytes) — `request_ts_ms` was used locally to compute the signature and then silently dropped from the wire frame. `cmd/provider/handler_repair.go`'s own "WIRE-FORMAT CORRECTION" comment already flagged the fix on the receiving side: the responder cannot verify `repair_auth_sig` (signed over `request_ts_ms`) without also receiving that field, and separately cannot freshness-check the request per ADR-036 §2 without it. The provider required and read the corrected 104-byte frame while the client kept sending the pre-correction 96-byte one — every repair-download request was rejected (length mismatch, stream reset) before Frame 2 was ever sent, indistinguishable from every holder being unreachable.
**Fix:** Frame 1 extended to `length(4) ‖ chunk_id(32) ‖ request_ts_ms(8) ‖ repair_auth_sig(64)`, matching the provider's corrected 104-byte frame.
**Confirmed live:** repair-download requests are now correctly authorised and signature-verified — status progressed from a stream-reset/length-mismatch to a clean `0x01 NotFound` response, proving the frame parses correctly end-to-end (the `NotFound` itself was F-16-4, below).

### F-16-4 — `findSurvivingHolders` never selected `chunk_id`, so every download requested the wrong shard

`cmd/microservice/repair_loop.go`'s `findSurvivingHolders` queried `chunk_assignments` for `provider_id, shard_index` only — never `chunk_id`. Every `SurvivingHolder` therefore carried a zero-value `chunk_id`, and `ExecuteRepairJob` fell back to using `job.ChunkID` (the *lost* shard's own content hash) as the download request against **every** surviving holder. Each surviving holder genuinely holds different bytes at a different shard index — a different SHA-256 content address, by construction of RS systematic/parity encoding — so every request deterministically returned `repairDownloadStatusNotFound`, from every holder, every time. This fully explained the "0 of 3 recovered" failures seen in every run once F-16-3 stopped masking it behind a transport-level error.
**Fix:** `SurvivingHolder` gained a `ChunkID [32]byte` field, populated per-holder from `chunk_assignments.chunk_id`; `downloadShards`/`downloadOneShard` request each holder's own chunk, not a single shared parameter. `job.ChunkID` is untouched everywhere else — RS re-encoding is deterministic, so the replacement shard is re-uploaded under its original identity.
**Confirmed live:** compiled, race-clean unit suite passing; download requests correctly matched to the actual holder's chunk at time of this write-up (full end-to-end confirmation blocked on F-16-5, below, which sits earlier in the same pipeline).

### F-16-5 — `findMissingShardIndex` derived the missing shard index by elimination, which breaks under concurrent multi-shard departure

Once F-16-3 and F-16-4 let a repair job reach the reconstruct stage, `ExecuteRepairJob` called `findMissingShardIndex(survivingHolders, profile.TotalShards)` — inferring "the missing index" by scanning which `ShardIndex` was absent from whatever `survivingHolders` list the caller supplied, and erroring unless exactly one index was absent. That function's own doc comment already named the assumption: "this package's repair pipeline handles exactly one missing shard per job." The assumption is correct about how many shards *this job* is responsible for — it is not correct about how many shards happen to be simultaneously non-`ACTIVE` in the segment. `findSurvivingHolders` filters `WHERE status = 'ACTIVE'`; the moment a **second** shard of the same segment is also non-`ACTIVE` (two providers departing close together, each independently spawning its own repair job), that filter excludes both missing shards from *every* job's holder list, not just the one a given job is repairing. Elimination then finds two gaps where it expects exactly one.

Live verification reproduced this directly: `TestDemoTimeline` (one departure) failed on a *different*, unrelated error — see F-16-6 below — but `TestViabilityRepairSucceedsWithTwoOfFiveOffline` (mvp.md §7.2's two-simultaneous-departure boundary case) failed all four of its spawned repair jobs identically: `findMissingShardIndex: want exactly one missing index among 3 shards (TotalShards=5), found 2` — exactly the shape predicted (2 departed providers × 2 segments = 4 jobs, each seeing `TotalShards-2` present holders instead of `TotalShards-1`).

**Fix:** replaced elimination with a direct lookup. `job.ChunkID` is a content hash, and RS re-encoding is deterministic (`ExecuteRepairJob`'s own doc comment) — a given shard index hashes to the same `chunk_id` for the life of a segment, repair after repair. `shard_index` is therefore a fact intrinsic to `job.ChunkID` itself, recorded once at original assignment time and never contradicted afterward, including on the departed holder's own row — soft-deleted, never hard-deleted (`departure.go`'s soft-delete discipline, M9 review Finding #1; confirmed no `DELETE FROM chunk_assignments` exists anywhere in the codebase). The new `lookupShardIndexForChunk` queries `chunk_assignments` directly by `chunk_id`, regardless of status or how many other shards of the segment are currently missing. `findMissingShardIndex` is removed; `setupFullPipelineFixture` (executor_test.go) updated to leave the real soft-deleted row behind, since the new lookup depends on it; a direct regression test (`TestRepairExecutorMissingIndexNotDerivedFromHolderCount`) added, matching F-16-3's own direct-regression-test precedent.
**Verification status:** compiles clean, gofmt-clean, and type-checks against a standalone stdlib-only reproduction of the new function. **Not yet confirmed against live Postgres / real daemons** — this sandbox has neither, and the module graph's transitive `prometheus/client_golang` dependency (via `internal/scoring → internal/audit`) requires a newer Go toolchain than is available offline here. Next live run should confirm `TestViabilityRepairSucceedsWithTwoOfFiveOffline` gets past this specific error — see F-16-6 for what it is expected to hit immediately afterward.

## Decision

The repair pipeline's wire protocol (F-16-3), holder-to-chunk binding (F-16-4), and missing-index derivation (F-16-5) are corrected, together with the adjacent file-registration signing bug (F-16-1) that was blocking the timeline one step earlier. Going forward, the same standing principle ADR-070 established applies here without modification: a pipeline stage is not considered verified until it has been run live against its real neighbour, not merely unit-tested against a mock of it.

## Consequences

**Closed by this ADR, confirmed live:** F-16-1, F-16-3 (partially — see F-16-4 for the remaining half of the same symptom), F-16-4 (unit-level; full live confirmation pending F-16-5).
**Closed by this ADR, not yet live-confirmed:** F-16-5.

**Still open, explicitly, not silently worked around:**

- **F-16-6 — repair-replacement provider selection structurally cannot succeed at the demo's fixed 5-provider topology**, independent of any of the fixes above. See its own ADR-075 (Proposed) — this is a distinct finding from F-16-1 through F-16-5 and is not a code defect in the same sense; it requires a decision, not a patch.
- `TestDemoTimeline`'s one-departure repair scenario has never yet completed end-to-end — it currently fails on F-16-6, not on any finding in this ADR.
- `TestViabilityActiveTransitionAtTenMinutes`'s timing window (5m30s±2min) has shown one out-of-window result (7m30s) against otherwise-passing runs; noted in the prior handover as likely scheduling jitter under load, not chased further pending confirmation this remains true across repeated runs.

## Verification

- `go build ./...`, `go vet ./...`, `gofmt` clean across all touched files (`internal/api/file.go`, `internal/api/file_test.go`, `internal/repair/executor.go`, `internal/repair/executor_test.go`, `internal/repair/repair_test.go`, `internal/repair/departure.go`, `cmd/microservice/repair_loop.go`).
- `go test -race -count=1 -p 1 ./...`: full suite passing, including `internal/repair`'s live-Postgres-backed package tests, as of the run immediately preceding F-16-5's fix.
- F-16-1: confirmed live — file registration succeeds as the terminal step of a real upload.
- F-16-3: confirmed live — repair-download requests parse and authorise correctly (frame no longer rejected at the transport level).
- F-16-4: confirmed at the unit level (race-clean); full live confirmation is blocked on F-16-5 landing, since F-16-5 sits one step earlier in the same job's execution.
- F-16-5: implemented, gofmt- and syntax-verified; live confirmation is the next concrete action, per the note above.

## References

- ADR-070 — Provider Onboarding Lifecycle: Live-Verification Findings (the `F-070-N` precedent this ADR's `F-16-N` findings follow the same discipline as)
- ADR-073 — Initial Upload Assignment: Client-Submitted, Per-Segment Content-Hash `chunk_id` (the decision this ADR's live-verification effort continues past)
- ADR-036 §2 — request freshness / `AuthRequestFreshnessWindow` (F-16-3)
- IC §4.4.1 — Repair-Download Stream Protocol (F-16-3, F-16-4)
- IC §3.2 — signing-input composition (`SignBytes`/`VerifyBytes`, hash-then-sign) (F-16-1)
- mvp.md §7.2 — RS(3,5) reconstruction math (F-16-5's reproduction case, `TestViabilityRepairSucceedsWithTwoOfFiveOffline`)
- M9 review Finding #1 — `chunk_assignments` soft-delete discipline (the invariant F-16-5's fix relies on)
