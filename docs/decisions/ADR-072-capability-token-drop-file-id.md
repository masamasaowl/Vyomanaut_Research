# ADR-072 — Capability Token: Drop `file_id`, Not Add It to the Wire Format

**Status:** Accepted
**Track:** DEMO
**Topic:** #14 Client Interface / #13 Provider Daemon Core (capability-token verification)
**Supersedes:** — *(clarifies IC §4.1's signing-input formula; the wire format itself is unchanged)*
**Superseded by:** —
**Research source:** none — this ADR records a Design Council resolution to a question ADR-070 (F-070-9) explicitly flagged as needing one.

---

## Context

ADR-070 (F-070-9) closed the `provider_id` half of a capability-token verification gap live-found in Session 16.1.1, but left the `file_id` half explicitly open: IC §4.1's `capability_token` signing input was specified as `domain_prefix || chunk_id(32) || provider_id(16) || file_id(16) || expiry_unix_ms(8)`, but IC §4.1's `UploadRequest` wire format (`chunk_id`, `shard_index`, `capability_token`, `chunk_data`) never carried a `file_id` field at all — a provider daemon had no way to learn the real value. Every real (non-vetting) upload failed capability-token verification (`0x03 NOT_ASSIGNED`) as a result; only vetting chunks (always minted with `file_id = uuid.Nil`) verified correctly, by coincidence of the zero value matching what the daemon could construct.

Confirmed live once the demo timeline could finally reach a real (post-`ACTIVE`) upload attempt (ADR-071): every real upload failed this exact check.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **(A)** Add `file_id` to `UploadRequest`'s wire format | Matches IC §4.1's formula as originally written, no formula change | Real protocol change: OAS, IC, the client SDK's frame construction, and the provider daemon's frame parsing all need updating, for a field that may add no actual protection |
| **(B)** Drop `file_id` from the signing input entirely — chosen | No wire-format change; smaller signing input in three already-identified minting call sites plus the one verification site; every structural check (length, expiry, `chunk_id` content-hash binding) is unaffected | Requires confirming `file_id` was never load-bearing — a security claim that needs checking, not assuming |

## Decision

**Drop `file_id` from `capability_token`'s signing input entirely.** IC §4.1's formula is corrected to `domain_prefix || chunk_id(32) || provider_id(16) || expiry_unix_ms(8)`.

The Design Council's security case, verified against the live implementation before this decision was made: `chunk_id` is 256 bits of fresh, microservice-generated randomness, minted exactly once per assignment (`internal/api/upload.go`'s `assignSegment`/`respondWithFreshTokens`, `internal/repair/executor.go`'s replacement-assignment path, `internal/vettingchunk/generator.go`'s synthetic-chunk path), and never reused across files. A capability token bound to a specific `chunk_id` is therefore already bound to a specific `(file, segment, shard)` assignment — `file_id` in the signing input protected nothing `chunk_id`'s own generation properties didn't already protect. No replay path exists that `file_id` would have closed and `chunk_id` alone leaves open.

**Implemented:**

- `internal/api/upload.go`'s `generateCapabilityToken` — `file_id` parameter and signing-input field removed; both call sites (`assignSegment`, `respondWithFreshTokens`) updated; `respondWithFreshTokens`'s now-unused `fileID` parameter removed from its own signature and its caller.
- `internal/repair/executor.go`'s `mintCapabilityToken` — same removal; its now-unused `fileIDForSegment` helper (whose only purpose was supplying this parameter) deleted entirely as dead code.
- `internal/vettingchunk/generator.go`'s `mintCapabilityToken` — same removal (this call site already passed `uuid.Nil` for `file_id`, so the removal has zero behavioral effect here — it only matters for the two paths above, which previously passed real, non-nil values).
- `cmd/provider/handler_upload.go`'s `capabilityTokenSigningInput`/`verifyCapabilityTokenFrame` — matching removal on the verification side; the file's own header comment (previously describing this as a known, open gap) is updated to record both halves of F-070-9 as closed.

## Consequences

**Closed:** F-070-9 in full (both the `provider_id` half from ADR-070 and the `file_id` half from this ADR). Capability-token verification no longer structurally rejects real uploads.

**Verified live:** confirmed against the real, running system — 9 of 10 shards across a 2-segment real file upload now verify and store correctly, the first time any real (non-vetting) shard data has ever been successfully stored in this codebase's history.

**Not closed by this ADR — a new, separate, deterministic finding:** the 10th shard (segment 1, shard 4 — the last shard of the file's second, partial segment) fails the same `0x03` check consistently and reproducibly, including under forced-sequential upload (`maxUploadConcurrency` temporarily set to 1 for diagnostic purposes only, confirming this is not a concurrency race — that hypothesis is ruled out, not just untested). The `chunk_assignments` row for this shard is well-formed (consistent `provider_id`, `ACTIVE` provider, matching the pattern of every other row). Root cause is not yet identified; the most promising unexplored lead is the client SDK's session-state population (`sess.ChunkIDs[segIdx][shardIndex]`, `internal/client/upload/transfer.go`) or the server's `assignSegment` handling specifically for a file's final, partial (non-full-length) segment — both untested hypotheses, not confirmed. Tracked as the next item for build.md Session 16.1.1's continuation, not resolved here.
