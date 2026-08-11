# ADR-073 — Initial Upload Assignment: Client-Submitted, Per-Segment Content-Hash `chunk_id`

**Status:** Accepted
**Track:** DEMO
**Topic:** #14 Client Interface / #11 Upload Assignment Service (capability-token binding)
**Supersedes:** — *(corrects `internal/api/upload.go`'s original Session 11.7.1 resolution of a self-flagged gap; does not reopen ADR-072, which stands unchanged)*
**Superseded by:** —
**Research source:** none — Design Council resolution to F-070-13 (ADR-070), raised once ADR-072 unblocked real uploads far enough to expose it.

---

## Context

F-070-13 (ADR-070) was opened as "one shard deterministically fails capability-token verification on multi-segment files" — 9 of 10 shards across a 2-segment test file reported as verifying correctly, with only segment 1/shard 4 failing.

Re-examination this session found the framing itself was wrong. `internal/api/upload.go`'s `assignSegment` (Session 11.7.1) mints `capability_token` bound to a `chunk_id` generated via `rand.Read` — chosen before the client has AONT/RS-encoded any content, a decision that session's own header comment already flagged as a compromise ("real content hashing can't produce this value in advance"). IC §4.1 defines `chunk_id` unconditionally as `SHA-256(chunk_data)` — the provider's own pre-write check enforces it (`0x02 CHUNK_ID_MISMATCH`), and storage, retrieval, audit, repair, and vetting are all built on it holding. `internal/client/upload` (unchanged since M15) independently computes the IC-correct content-hash `chunk_id` and has no field to receive the server's random one at all.

Given this, capability-token verification for a real upload cannot succeed under the current code for *any* shard — the two values are independent 256-bit randomness with no mechanism to coincide. This was verified by isolated reproduction (Ed25519 sign/verify against both candidate values, no repo dependencies) rather than assumed.

The "9 of 10 shards verified live" result recorded in ADR-072 cannot be reconciled with this: `git log --follow` confirms zero commits touched `internal/client/upload` since M15, before or after ADR-072. The previous session's own account: the "`chunk_id` is opaque randomness" framing was inherited from an M13-era code comment in `handler_upload.go` without checking it against IC §4.1 directly — if that comment was wrong, the error propagated into ADR-072's own reasoning too. This ADR does not reopen ADR-072's actual decision (dropping `file_id` from the signing input remains correct and unaffected); it corrects the separate, earlier assumption ADR-072 inherited about what `chunk_id` itself is allowed to be.

**Explicitly out of scope:** `internal/repair/executor.go` and `internal/vettingchunk/generator.go`. Both already mint capability tokens against a `chunk_id` that is a real, already-known content hash at mint time — repair reconstructs deterministically-identical bytes for an already-assigned `chunk_id`; vetting hashes its synthetic data before minting. Neither requires any change under this decision.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **(A)** Client encodes the whole file, then submits all real `chunk_id`s in one assignment call | Single call per file, matches today's endpoint shape exactly | Pays full AONT/RS encoding cost for the entire file before learning whether readiness/escrow would even allow the upload — a real regression at prod file sizes |
| **(A′)** Per-segment: client encodes segment *i*, submits *i*'s real `chunk_id`s, before encoding segment *i+1* — **chosen** | Preserves fail-fast: readiness/escrow are still checked (and can still reject) before segment 1 is ever encoded, matching today's actual cost profile; no Frame 1/2 wire-format change; `chunk_id` keeps one meaning everywhere | Requires the assignment endpoint to accept incremental, partial submissions for a file rather than an all-or-nothing call; existing idempotency logic (`loadExistingAssignments`/`respondWithFreshTokens`) must generalize from "resend" to "resend-and-extend" |
| **(B)** New two-phase endpoint: `/upload/assign` returns providers only; a second call confirms real `chunk_id`s and issues tokens | Cleanly separates "can this upload happen" from "here are the exact bytes" | New endpoint, new auth/idempotency surface, and an "unconfirmed" `chunk_assignments` row needs its own GC (like `audit_receipts`' PENDING-row GC) — a full extra session, not a same-session fix |
| **(C)** Decouple identity from content: introduce a server-assigned opaque `slot_id` for capability-token binding, keep `chunk_id` purely content-hash for storage/retrieval | Removes the "assign before content exists" tension by construction | Dead on arrival: Frame 1's byte layout (32+4+72+262144 = 262,252 bytes, IC §4.1) has no spare field for a second identifier without a protocol version bump; would also touch the download protocol, which currently addresses shards purely by `chunk_id` |

## Decision

**Adopt (A′).** `POST /api/v1/upload/assign` becomes callable incrementally, once per segment as the client finishes encoding it, carrying that segment's real, client-computed `chunk_id`s in the request body. The server assigns only the segments named in a given call that don't already exist for that `file_id`; readiness, escrow, and per-provider capacity checks run only on the first call for a `file_id` (extending, not replacing, the endpoint's existing "idempotent on `file_id`" ERRATA precedent); every response — first call or subsequent — returns the full, fresh-tokened assignment set persisted so far for that file, so the client's last call always has everything `finishUpload` needs to build the pointer file.

**`assignSegment`** stops calling `rand.Read` for `chunk_id` and instead takes the client-supplied value per shard directly. Capability-token minting is consolidated into a single call site (`respondWithFreshTokens`, already regenerating tokens on every response) rather than duplicated inside `assignSegment` — `assignSegment` now only creates DB rows.

**Wire format:** `UploadAssignRequest` gains a `segments` array (`segment_index`, `chunk_ids[TotalShards]` hex) as client-submitted input. Frame 1/2 of the chunk-upload stream protocol itself (IC §4.1) is **unchanged** — `chunk_id` remains exactly `SHA-256(chunk_data)`, 32 bytes, everywhere.

**No change** to `cmd/provider/handler_upload.go` (verification was already correct), `internal/repair/executor.go`, or `internal/vettingchunk/generator.go`.

## Consequences

**Closes F-070-13** — correctly re-scoped: this was never a single-shard edge case, and the fix restores an invariant `internal/api/upload.go` violated at M11, not a narrow retry-path bug.

**Corrects the record on ADR-072:** ADR-072's own decision (dropping `file_id`) stands. Its "9 of 10 shards verified live" empirical claim should be treated as unverified against the current, committed `internal/client/upload` — most likely produced by an ad-hoc/uncommitted test harness rather than the orchestrator now wired into `demo_timeline_test.go`. Not re-litigated further here; not blocking this decision either way.

**New idempotency responsibility:** `HandleAssign` must distinguish "segment already exists → refresh its token only" from "segment not yet submitted → create it" on every call, rather than the previous binary "any existing rows → skip everything." This is the real complexity this ADR accepts in exchange for keeping the wire protocol and the single-endpoint shape unchanged.

**Deferred, not blocking:** `mvp.md §3.6`'s demo timeline narrates assignment as a single instantaneous step before encoding; per-segment interleaving makes this imprecise at the sub-step level (not wrong at the T+01:00 granularity it's written at). A one-line footnote update is left for a future documentation pass, not this ADR.

## References

- IC §4.1 — Chunk Upload Stream Protocol (`chunk_id` definition, Frame 1/2 layout)
- ADR-070 — Provider Onboarding Lifecycle Live Verification (F-070-13's original filing)
- ADR-072 — Capability Token: Drop `file_id`, Not Add It to the Wire Format (the decision this ADR narrows the scope of the inherited assumption from, without reopening)
- Design Council session (five-seat verdict), this build continuation
