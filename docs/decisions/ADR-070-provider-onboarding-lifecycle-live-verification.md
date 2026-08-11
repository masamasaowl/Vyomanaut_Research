# ADR-070 — Provider Onboarding Lifecycle: Live-Verification Findings (Registration → Heartbeat → Vetting → Active)

**Status:** Accepted
**Track:** DEMO
**Topic:** #13 Provider Daemon Core, #16 Provider-Side Storage Engine (capability-token half only), #4 Microservice Coordination Loops
**Supersedes:** —
**Superseded by:** —
**Research source:** none — this ADR records implementation-gap findings from live end-to-end execution, not new external research.

---

## Context

Milestone 16, Session 16.1.1 requires `scripts/test/demo_timeline_test.go` to exercise the full demo timeline from `mvp.md` §3.6: readiness gate, upload, first audit pass, VETTING→ACTIVE transition, synthetic chunk GC, departure detection, and repair. Writing that test against unverified assumptions about how registration, heartbeat, and vetting actually behave would have produced a test that either passed for the wrong reasons or failed for reasons unrelated to the test file itself.

Before writing the test, the underlying system was run live: a real Postgres instance, a real `cmd/microservice` binary, and five real `cmd/provider` processes (via `--sim-count`/`--sim-only-index`, Session 16.2.1), driven through actual HTTP registration, actual OTP verification, actual heartbeats, actual synthetic chunk uploads, and actual audit challenges — not mocked, not stubbed.

**No provider, in any deployment of this codebase's history before this session, had ever completed the full onboarding lifecycle from registration to `ACTIVE`.** Ten distinct defects were found along that path, each blocking progress at a different stage. Every one of them was invisible to the existing test suite, because each component's unit tests correctly tested that component in isolation — never the seam where it hands off to the next stage. Several were self-documented as known gaps in code comments at the time they were written (Milestones 11–14), correctly describing preconditions that had not yet been met; this ADR records where those preconditions were finally met, and what remained broken even after they were.

## Findings

Each finding is labeled `F-070-N`, in the order it was hit during live verification (later findings were only reachable because earlier ones were fixed first).

### F-070-1 — Provider self-registration was entirely absent

No code anywhere in `cmd/provider` called `POST /api/v1/provider/register`. Without it, a provider never appears in `providers`, and the readiness evaluator's `active_vetted_providers`/`distinct_asns` conditions can never be satisfied regardless of how long a demo runs.
**Fix:** `registerProviderWithMicroservice` (`cmd/provider/main.go`), called from `runProviderInstance` once identity is established.

### F-070-2 — Registration signatures used the wrong composition

`provider_sig` was computed as `ed25519.Sign(signingKey, canonicalRegisterSigningInput(req))` — signing the raw canonical bytes directly. `internal/crypto`'s `SignBytes`/`VerifyBytes` (the actual verification path, IC §3.2) use a **hash-then-sign** composition: `SHA-256(input)`, then `Ed25519.Sign` on the digest. Every registration attempt failed with `INVALID_BODY_SIGNATURE`. This was not visible from comparing the canonical-input-construction functions side by side — `canonicalRegisterSigningInput`'s output was byte-correct; only the signing step over that output was wrong. Found by reproducing the request by hand outside Go (Python + OpenSSL + an independent Ed25519 implementation) and tracing the mismatch into `internal/crypto/ed25519.go` directly.
**Fix:** sign `sha256.Sum256(canonicalRegisterSigningInput(req))`, not the raw bytes.

### F-070-3 — Provider registration additionally required an OTP-issued bearer token

`POST /api/v1/provider/register` is wrapped in `bearerAny` (router.go) — undocumented in the registration flow's own description and not anticipated by F-070-1's original fix. A provider daemon has no legitimate way to complete OTP verification itself: it has no real phone number, and giving it direct database access to read its own OTP code (the only way to recover the code — see F-070-3a) would be architecturally wrong for an untrusted third-party process.
**Fix:** the token is obtained externally — by whatever legitimately orchestrates the daemon and already needs database access for other reasons (a test harness, an operator tool) — and supplied via a new `--registration-bearer-token` flag. `cmd/provider` never touches the database.

**F-070-3a (sub-finding):** the OTP code is stored only as a SHA-256 hash (`otp_codes.code_hash`); the plaintext is never persisted or logged, even by the demo-mode `NoopOtpSender`. A code comment describing "reading `otp_codes` directly in a demo/dev environment" as a way to recover the code is incorrect — the hash alone doesn't recover it. Brute-forcing the 6-digit space (1,000,000 SHA-256 hashes) against the stored hash does recover it, in well under a second, and is a legitimate approach for a test harness or dev operator with direct database access; it is not something `cmd/provider` itself should ever do.

### F-070-4 — Heartbeat sent no `Authorization` header at all

`POST /api/v1/provider/heartbeat` is `bearerAuthRole("provider")`-gated. `internal/p2p/heartbeat.go`'s `postHeartbeat` took no token parameter and set no `Authorization` header, even though `HeartbeatConfig` already declared `GetToken`/`RefreshToken` fields — nothing ever read `GetToken()`'s value into the request. Every heartbeat, from every provider, in every deployment, was rejected with 401 regardless of whether registration succeeded.
**Fix:** `postHeartbeat` now takes a `token string` parameter and sets `Authorization: Bearer <token>` when non-empty; `doHeartbeat` calls `cfg.GetToken()` to supply it. `cmd/provider/main.go` wires `GetToken` to the JWT captured at registration (F-070-1) and `RefreshToken` to a documented no-op (the JWT's 7-day TTL far exceeds any single daemon run in this codebase's current scope).

### F-070-5 — Heartbeat's `provider_sig` was base64-encoded; the server expects hex

Already self-documented as a known, unfixed cross-package bug in `internal/api/provider.go`'s header comment at the time it was written — flagging that `internal/p2p/heartbeat.go`'s `doHeartbeat` and `HandleHeartbeat`'s `decodeProviderSig` disagreed on wire encoding, and that fixing the client side would require updating `internal/p2p/heartbeat_test.go`'s own assertion of the old (wrong) encoding.
**Fix:** `hex.EncodeToString(sig[:])`, matching every other `provider_sig` in the system; `heartbeat_test.go`'s assertion updated to decode hex instead of base64.

### F-070-6 — `heartbeatCfg.ProviderID` carried the wrong identity entirely

Set to `string(peerID)` — the libp2p Peer ID — not the microservice-assigned UUID. `HandleHeartbeat` compares this field against the authenticated JWT's `sub` claim (a UUID) for an exact match; a Peer ID string could never match. This would have silently defeated F-070-4's fix even after it landed.
**Fix:** `ProviderID` is now the UUID returned by registration (F-070-1).

### F-070-7 — Heartbeat never reported the daemon's own address

`heartbeatCfg.CurrentAddrs` was hardcoded to `func() []p2p.Multiaddr { return nil }`, documented at the time as an out-of-scope NAT/relay-discovery gap. Consequence: `providers.last_known_multiaddrs` stayed `NULL` even after heartbeat itself started succeeding (F-070-4–6), so nothing could ever dial a provider back — including the vetting-chunk generator (F-070-8).
**Fix:** for the local/simulation context this codebase currently runs in (no NAT involved — every simulated provider binds to `127.0.0.1`), `CurrentAddrs` now reports the daemon's own known local listen address, the same one already used for registration. **Real NAT traversal / relay address discovery for non-local deployments remains open** — this fix is honest and correct for loopback, not a general solution.

### F-070-8 — The synthetic vetting-chunk generator was never wired into `cmd/microservice`

`internal/vettingchunk`'s `Generator` (Milestone 14) was never imported anywhere in `cmd/microservice`. Without a `chunk_assignments` row, `runAuditDispatchLoop`'s `loadActiveChunkAssignments` always returns empty for a VETTING provider, so nothing is ever audited, and `consecutive_audit_passes` can never advance past 0 — VETTING→ACTIVE was unreachable regardless of every other fix in this ADR.
**Fix:** new `cmd/microservice/vetting_chunk_loop.go`, wired into `main.go` as an appended "Step 19" (the original 18-step sequence is left renumbered-free). Generates a small fixed target (3 chunks) per VETTING provider per cycle — not `Generator.Cap()`'s production-scale ceiling (`declared_storage_gb × 400`, e.g. 40,000 for a 100 GB demo provider), which bounds how many a provider could ever be assigned, not a target this loop tries to reach.

### F-070-9 — Capability-token verification used an always-zero `provider_id`

Already self-documented in `cmd/provider/handler_upload.go`'s header comment at the time it was written: capability-token verification needs the daemon's own microservice-assigned `provider_id`, which did not exist anywhere in scope at Milestone 13 — every call site passed the zero UUID, so every real capability token failed verification with `0x03 NOT_ASSIGNED`, by design, fail-closed. F-070-1 finally supplies the real value.
**Fix:** the previously always-zero `providerID` struct field is removed; capability-token verification now uses the same `providerIDBytes` value already used for receipt signing, computed in `main.go` from the real registered UUID (parsed via `uuid.Parse`) once registration succeeds, with a documented fallback to the prior Peer-ID-derived placeholder otherwise.
**What this does not close:** the signing input's `file_id` field stays the zero UUID unconditionally — correct for synthetic vetting chunks (`internal/vettingchunk` always signs with `file_id = uuid.Nil`, matching DM §8.20's documented nil-for-vetting semantics), but still wrong for real (non-vetting) uploads, whose capability tokens carry a real, non-nil `file_id` the daemon has no way to learn: IC §4.1's `UploadRequest` Frame 1 has no `file_id` field at all. Real uploads still fail this check. Closing that half requires either a wire-format change or dropping `file_id` from the signing input entirely (already flagged in the original header comment as "a call for the design council").

### F-070-10 — `providers.first_chunk_assignment_at` was never written, anywhere

`internal/scoring/passes.go`'s `IncrementConsecutivePasses` already has a careful, defensive `sql.NullTime` check against this column, with a comment explicitly describing it as set by "the assignment service" on a provider's first chunk assignment. No code anywhere — not `internal/vettingchunk`, not any `cmd/microservice` file — ever wrote it. Consequence: `VettingMinDuration`'s gate (`first_chunk_assignment_at + 5m <= NOW()`) could never be satisfied no matter how many consecutive audit passes accumulated (confirmed live: `consecutive_audit_passes` reached 6, well past `VettingMinPasses:5`, while every provider stayed `VETTING`, invisibly, indefinitely).
**Fix:** `vetting_chunk_loop.go` (F-070-8) sets it via an idempotent `UPDATE providers SET first_chunk_assignment_at = NOW() WHERE provider_id = $1 AND first_chunk_assignment_at IS NULL`, immediately after a provider's first successful synthetic chunk upload.

### F-070-11 — The cluster audit secret's cache was never refreshed

`internal/audit.ClusterSecretCache` has an intentional 5-minute TTL (IC §8) — by design: a real deployment must periodically re-validate the cluster audit secret against its secrets manager. `cmd/microservice/main.go` called `cache.Load(ctx)` exactly once, at startup, with no periodic re-call anywhere. Consequence: every audit dispatch succeeded for the cache's first 5 minutes, then every subsequent dispatch attempt failed closed with `ErrSecretExpired` for the rest of any run — regardless of the underlying secrets client (`envSecretsClient`, itself correctly implemented and always available) — because nothing ever asked it again. This has no startup-time symptom: `readiness.go`'s `cluster_audit_secret_loaded` condition checks only whether the cache was *ever* loaded, not whether it is still fresh, so a demo run can look fully green at minute 1 and have every audit silently stop dispatching by minute 6.
**Fix:** new `cmd/microservice/cluster_secret_refresh_loop.go`, wired into `main.go` as "Step 20", re-calling `cache.Load` every 2 minutes (comfortably under the 5-minute TTL).

### F-070-12 — Vetting GC delivery was never wired in either

`internal/vettingchunk.GCDelivery.DeliverGCInstruction` (Milestone 14) — the call that removes a provider's synthetic vetting chunks on its `VETTING`→`ACTIVE` transition (IC §5.10) — was never called anywhere in `cmd/microservice`, the same shape of gap as F-070-8. The reason is structural: `internal/scoring.IncrementConsecutivePasses` performs the database transition but returns only `error`, with no way to signal "a transition just happened" to its caller (`runAuditDispatchLoop`). Nothing was ever positioned to react to the one moment IC §5.10 specifies.
**Fix:** new `cmd/microservice/vetting_gc_loop.go`, wired into `main.go` as "Step 21" — rather than changing `IncrementConsecutivePasses`'s signature (an already-tested package; this fix stays entirely inside `cmd/microservice`, matching F-070-8's and F-070-11's discipline), it polls for the resulting state instead of the event: any `ACTIVE` provider that still has live (`status = 'ACTIVE'`) synthetic `chunk_assignments` has not yet had GC delivered, regardless of how long ago the transition happened.
**Verification status:** fixed and confirmed via the isolated test suite (builds clean, `go vet` clean, zero regressions) — **not yet observed succeeding in a completed live end-to-end run**, since the run that would exercise it was blocked earlier in the timeline by the finding ADR-071 records (upload attempted before any provider reached `ACTIVE`). Once ADR-071's test re-sequencing lands, this is the next thing to confirm live.

## Decision

The full chain — registration, heartbeat authentication, heartbeat address reporting, capability-token verification, vetting-chunk generation, vetting-duration tracking, vetting GC delivery, and cluster-secret freshness — is wired correctly. Eleven of twelve findings are confirmed live end-to-end: five real, independently-launched provider processes registered, heartbeated, transitioned `PENDING_ONBOARDING`→`VETTING`, received and uploaded real synthetic chunks, accumulated audit passes, and transitioned `VETTING`→`ACTIVE`, entirely through the real HTTP/wire protocols this codebase specifies — no step mocked, stubbed, or bypassed. F-070-12 (GC delivery) is fixed and unit-verified but awaits its own live confirmation, blocked on ADR-071's test fix landing first.

Going forward: **a milestone that adds a new stage to a multi-stage provider lifecycle must be verified against the stage immediately before and after it, live, before the milestone is considered complete** — not merely unit-tested in isolation. Twelve of twelve findings in this ADR were exactly this shape: a component correctly implementing its own documented contract, defeated by a neighboring component that either didn't exist yet or disagreed with it at the boundary. Unit tests, by construction, cannot catch a disagreement between two components; only running the seam can. See ADR-071 for a thirteenth instance of the same underlying pattern, one level up: a *planning document* disagreeing with an already-ratified decision (ADR-030) it was never reconciled against.

## Consequences

**Closed by this ADR**, confirmed live: F-070-1 through F-070-11. **Closed but not yet live-confirmed:** F-070-12. **F-070-9 fully closed** (both provider_id and file_id halves) via ADR-072, which also surfaced F-070-13 — a new, distinct, deterministic finding — not yet resolved.

**Still open, explicitly, not silently worked around:**

- ~~**F-070-9's file_id half**~~ — **CLOSED by ADR-072** (Design Council #2): dropped `file_id` from the capability-token signing input entirely rather than adding it to the wire format, since `chunk_id`'s own generation properties (fresh 256-bit randomness, never reused across files) already provide the binding `file_id` was meant to. See ADR-072 for the full verdict and F-070-13 below for what surfaced immediately once this unblocked real uploads.
- **F-070-13 (new) — one shard deterministically fails capability-token verification on multi-segment files.** Once ADR-072 unblocked real uploads, 9 of 10 shards across a 2-segment test file verified and stored correctly — the first real shard data ever successfully stored in this codebase's history. The 10th (segment 1, shard 4 — the last shard of the file's final, partial segment) fails `0x03` deterministically and reproducibly, including under forced-sequential upload (ruling out a concurrency race, not merely leaving it untested). The `chunk_assignments` row is well-formed and consistent with every other row. Root cause not yet identified; most promising unexplored leads: the client SDK's session-state population (`internal/client/upload/transfer.go`'s `sess.ChunkIDs[segIdx][shardIndex]`) or the server's `assignSegment` handling specifically for a file's final, partial segment. **Not fixed. Next item for Session 16.1.1's continuation.**
- **F-070-7's general case** — real NAT traversal / relay address discovery for non-local, non-loopback deployments.
- **Provider registration has no retry loop** — a transient microservice outage at startup leaves a provider permanently unregistered until manually restarted.
- **Heartbeat token refresh is a no-op** — safe given the 7-day JWT TTL relative to this codebase's current run durations, but a real `POST /api/v1/provider/token/refresh` call is unimplemented.
- **Real ASN detection** — every registration in this codebase's current form is a simulation-mode registration (`demo_asn`); production ASN detection (`asn` field) is unimplemented.
- **`--sim-count`'s single-process design cannot register more than one provider** — registration tokens are single-use and tied to one phone-derived subject; running N providers as one process sharing one `--registration-bearer-token` can only ever register the first to reach that code path. `--sim-only-index` (Session 16.2.1) — running N separate OS processes, each with its own token — is required for correct multi-provider registration, not merely for independent kill/departure testing as originally scoped.
- **F-070-12's live confirmation** — see Verification status above.

## Verification

Live, end-to-end, five real provider processes against a real Postgres instance and a real `cmd/microservice` binary:

- All 5 registered (`POST /api/v1/provider/register`, real Ed25519 signatures, real OTP-derived bearer tokens).
- All 5 heartbeated successfully and transitioned `PENDING_ONBOARDING`→`VETTING` within one heartbeat interval.
- All 5 received and uploaded 3 real synthetic vetting chunks each (15 total), each via the real `/vyomanaut/chunk-upload/1.0.0` protocol with a verified capability token.
- All 5 accumulated audit passes via the real `/vyomanaut/audit-challenge/1.0.0` protocol, past `VettingMinPasses:5`, with continuous cluster-secret availability across the full run (zero `ErrSecretExpired` occurrences).
- All 5 transitioned `VETTING`→`ACTIVE`.
- Readiness gate (`GET /api/v1/admin/readiness`) reported `all_conditions_met: true` throughout, with `active_vetted_providers: 5/5` reflecting genuinely active providers by the run's end.
- Zero errors of any kind, on either the microservice or provider side, across the full run once F-070-1 through F-070-11 were fixed.
- F-070-12 (GC delivery): fixed, isolated-suite-verified, live confirmation pending — see its own entry above.

Every fix listed above additionally passed: `gofmt`, `go vet` (native and `GOOS=windows` where applicable), and the full existing test suite for every touched package (`cmd/provider`, `cmd/microservice`, `internal/p2p`) with zero regressions, verified in an isolated module mirroring each package's complete real dependency graph.
