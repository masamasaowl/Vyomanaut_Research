# ADR-074 — Addendum A: Confirm-Step MarkJobComplete Gap (F-16-7)

**Applies to:** [ADR-074 — Repair Pipeline Live-Verification Findings](./ADR-074-repair-pipeline-live-verification-findings.md)
**Status of parent ADR:** Accepted — **unchanged in intent.** ADR-074's own stated purpose is exactly this: repair-pipeline findings surfaced by continued live verification of the same demo timeline, in the same file (`internal/repair/executor.go`). This addendum adds the fifth finding to that same corpus, found one live run after the parent ADR's own F-16-5 was confirmed working.
**Added research source:** none — live-verification finding, same discipline as the parent ADR.
**Findings addressed:** F-16-7 (new numbering continues directly from the parent ADR's F-16-1 through F-16-5; F-16-6 is ADR-075's topology finding, a distinct decision, not part of this numbering line).

> **Insert location:** after the parent ADR's F-16-5 entry, as a sixth finding in the same "Findings" section; the parent's own "Decision" and "Consequences" sections are otherwise unchanged.

---

## Why this addendum exists

With F-16-5 (missing-index derivation) and F-16-6 (repair-replacement headroom, ADR-075) both confirmed live — `TestDemoTimeline` and `TestViabilityRepairSucceedsWithTwoOfFiveOffline` both passing clean — a subsequent live run of the identical, unchanged test suite produced:

```
demo_timeline_test.go:973: fewer than 1 repair jobs completed within 5m0s (completed=0, failed=0)
```

`pollDeparted` had already succeeded (departure detection worked); the failure is entirely inside `pollRepairCompleted`'s own 5-minute window, with `completed=0` **and** `failed=0` — the repair job never reached *either* terminal state. Nothing in the microservice log explained it: no `[REPAIR]` error line, because nothing on the path that actually failed ever logs or marks anything.

This is not a regression from F-16-5 or ADR-075's headroom fix — neither touched this code path — and it did not reproduce on the immediately following run (`TestDemoTimeline` passed clean in the same session's next invocation, alongside all four `TestViability*` tests). That shape — intermittent, no corresponding error output, same code on both the passing and failing run — is the signature of a genuine control-flow gap that only manifests under real concurrent DB load, not a deterministic logic error the previous four findings all were.

## F-16-7 — `ExecuteRepairJob`'s Confirm step never marked the job terminal on `activateChunkAssignment` failure

Every other error-return in `ExecuteRepairJob` (download, decode, re-encode, missing-index lookup, replacement selection, pre-register, upload, exhausted-retries) calls `MarkJobComplete(ctx, db, job.JobID, false)` before returning — a consistent, checked pattern across the whole function. Step 4 (Confirm) was the one exception:

```go
if err := activateChunkAssignment(ctx, db, job.ChunkID, replacementProviderID); err != nil {
    return fmt.Errorf("repair.ExecuteRepairJob: activate: %w", err)  // no MarkJobComplete
}
```

`activateChunkAssignment` is a single `UPDATE chunk_assignments SET status = 'ACTIVE' WHERE chunk_id = $1 AND provider_id = $2 AND status = 'REPAIRING'` — reached only *after* a real shard has already been uploaded and confirmed by the replacement provider. By this point in the demo timeline, several independent background loops are hitting the same Postgres instance concurrently: `runAuditDispatchLoop` (2 min ticker), `runVettingChunkGenerationLoop` (30 s ticker), the silent-departure detector (30 s ticker), and `runRepairExecutorLoop` itself (tight 2 s idle-backoff loop, potentially processing *other* jobs concurrently against the same tables). `chunk_assignments` is the single most contended table in the schema across all of them. A transient failure here — lock contention, a Postgres-resolved deadlock, a momentary connection hiccup — is exactly the class of failure `SelectReplacementProvider`'s bounded retry and the upload loop's `STORAGE_FULL` retry already treat as recoverable elsewhere in this same function. This one path didn't.

Without a `MarkJobComplete` call, the job stayed at `IN_PROGRESS` — set by `DequeueNextJob`'s own `markInProgress` UPDATE — permanently. `DequeueNextJob` only ever selects `WHERE status = 'QUEUED'`, so an `IN_PROGRESS` job is never retried by anything. It is invisible to `pollRepairCompleted`'s two queries (`COUNT(*) WHERE status = 'COMPLETED'`, `COUNT(*) WHERE status = 'FAILED'`), which is exactly the `completed=0, failed=0` symptom observed.

**Fix:** `activateChunkAssignment`'s error path now calls `MarkJobComplete(ctx, db, job.JobID, false)` before returning, matching every other path in the function.

**A second, related gap in the same step:** the function's *final* line, `MarkJobComplete(ctx, db, job.JobID, true)` itself, had the identical structural gap one call earlier — if that specific write failed, the function returned the bare error with nothing to fall back on. This one needed different handling, not the same pattern: by the time this call runs, the shard is already uploaded and `ACTIVE` — the repair has already succeeded. Falling back to `MarkJobComplete(..., false)` here would misreport a successful repair as a failure. Since a `FAILED` job is terminal (`runRepairExecutorLoop`'s own doc comment: the executor does not retry a job it has already marked `FAILED`), a false `FAILED` record would sit permanently wrong with no mechanism to correct it. This write has no external dependency once activation has already committed, so a transient failure is the only realistic cause, and a bounded retry (`markCompleteFinalWriteRetries = 3`, `markCompleteFinalWriteBackoff = 500ms`) is the correct shape — it either succeeds within a few attempts, or returns an explicit error naming the inconsistent state (shard active, job status unrecorded) rather than leaving it silently stuck.

## Verification

- `gofmt`-clean, syntax-verified (this sandbox cannot run a full `go build`/`go test` against live Postgres — same limitation as every other fix this session; needs live confirmation on the next run).
- No dedicated automated regression test was added for this specific path, and that gap is being named rather than papered over: triggering `activateChunkAssignment`'s real error branch requires an actual Postgres-level failure (a bad connection, a resolved deadlock, a constraint violation) — a `WHERE` clause matching zero rows does *not* produce a Go error from `database/sql`, so the ordinary unit-test technique of manipulating fixture data doesn't reach this branch at all. This codebase has no DB fault-injection layer to construct one deterministically. Consistent with this project's own standing principle — live verification over unit tests, because wire-protocol and concurrency bugs consistently evade the latter — the next live `TestDemoTimeline` run (ideally several, given the intermittent nature of what this addendum fixes) is the actual confirmation, not a new test function.

## References

- Parent ADR-074, F-16-5 (the prior finding in the same file, same function, confirmed live one run before this one surfaced)
- ADR-075 (F-16-6, the headroom finding — independent of this one; both live in `internal/repair`'s repair-replacement path but at different steps)
- `internal/repair/queue.go` — `DequeueNextJob`, `MarkJobComplete`, `EnqueueJob` (the terminal-state machinery this finding closes a gap in)
- `cmd/microservice/repair_loop.go` — `runRepairExecutorLoop` (the caller; its own catch-all `log.Printf` on error is unchanged by this fix, since `ExecuteRepairJob` now reliably marks terminal status on every path itself)
