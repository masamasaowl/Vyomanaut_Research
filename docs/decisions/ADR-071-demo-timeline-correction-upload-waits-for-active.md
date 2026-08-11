# ADR-071 — Demo Timeline Correction: Upload Waits for ACTIVE, Per ADR-030

**Status:** Accepted
**Track:** DEMO
**Topic:** #5 Peer Selection (vetting trust boundary) / #14 Client Interface (demo timeline)
**Supersedes:** — *(amends `mvp.md` §3.6 only; ADR-030's Decision is unchanged and unaffected)*
**Superseded by:** —
**Research source:** ADR-030 (Synthetic Vetting Chunks: Repair-Safe Provider Assessment); live end-to-end verification of `scripts/test/demo_timeline_test.go` (build.md Session 16.1.1)

---

## Context

Live verification of the demo timeline (Session 16.1.1, driving a real `cmd/microservice` and five real `cmd/provider` processes end-to-end, not mocked) found a genuine contradiction between `mvp.md` §3.6's stated demo timeline and the shipped implementation:

`mvp.md` §3.6 places the data owner's file upload — "5 shards placed" — at **T+01:00**, nine minutes before any provider transitions to `ACTIVE` (T+10:00 in the same timeline). The implementation disagrees consistently, in two independent places:

- `internal/api/upload.go`'s `eligibleActiveProviderCountAtOrUnder`: `WHERE p.status = 'ACTIVE'` — rejects the entire upload request with HTTP 503 unless `MinActiveProviders` providers are strictly `ACTIVE`.
- `internal/repair/assignment.go`'s `drawTwoActiveCandidates` (shared between initial shard assignment and later repair-replacement selection): "draws up to two random **ACTIVE** providers."

Confirmed live: the general readiness gate (`GET /api/v1/admin/readiness`) reports `all_conditions_met: true` — including `active_vetted_providers`, which counts `status IN ('VETTING', 'ACTIVE')` — while the upload-specific capacity check, evaluated 19 milliseconds later against the same database, still rejects the request. The two checks are not in a race; they are consistently, deliberately asking different questions, and only one of them matches `mvp.md` §3.6's stated timeline.

The document also contradicts itself internally: its own T+10:30 line reads "Real data owner shard assignments **begin**" — which only makes sense if they had not begun before.

This is not a live design tradeoff. **ADR-030 (Accepted) already settled it.** ADR-030 replaced an earlier design — ADR-005's "real shards on vetting providers, extra redundancy" — specifically because that design "weakens the durability guarantee during the vetting window," and because a vetting-period departure (the event vetting exists to screen for) would otherwise trigger a real repair cycle at exactly the network's most fragile phase. The implementation correctly carries ADR-030 forward; `mvp.md` §3.6 does not, most likely because the timeline sketch predates ADR-030 or was never reconciled with it once ADR-030 landed.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Loosen `eligibleActiveProviderCountAtOrUnder`/`drawTwoActiveCandidates` to also accept `VETTING` providers | Matches `mvp.md` §3.6's stated T+01:00 timing without editing the doc | Reopens exactly the durability/repair-bandwidth risk ADR-030 was ratified to close. `drawTwoActiveCandidates` is shared with repair-replacement selection — this is not a demo-scoped change, it is a production trust-model change wearing a demo justification. No production deployment should ever place real customer data on a provider with zero audit history |
| Leave both `mvp.md` and the code as-is; treat the discrepancy as a known test-ordering quirk | Zero changes | The next reader of `mvp.md` inherits a false timeline; the next integration test (or a live presenter reading the demo script) breaks the same way this session did. Silent, not fixed |
| **Correct `mvp.md` §3.6; re-sequence the integration test to match — chosen** | Zero production-code risk; ADR-030's trust boundary stays exactly as ratified; the fix is a doc edit plus a test re-ordering, both already within reach (the test's own `pollAllProvidersActive` helper already exists) | The corrected timeline has an idle-looking 9-minute window before the first successful upload; needs an explanatory line so a live audience isn't left wondering why nothing visibly happens |

## Decision

**1. `internal/api/upload.go` and `internal/repair/assignment.go` are unchanged.** ADR-030's `ACTIVE`-only trust boundary for real shard assignment is correct and stays exactly as ratified, for both demo and production profiles.

**2. `mvp.md` §3.6 is corrected.** The T+01:00 line is replaced to describe what the system actually does at that point — a rejected upload attempt, not a successful one:

```diff
- T+01:00  — Data owner uploads a test file (< 1.25 MB per segment; 5 shards placed)
+ T+01:00  — Data owner registers and deposits escrow; first upload attempt
+            is rejected (HTTP 503, providers still VETTING) — client retries
+            automatically per IC §5.9's documented backoff
```

T+10:30's existing line is unchanged — it was already correct and needs no edit:

```
T+10:30  — Vetting GC instruction delivered; synthetic chunks deleted
T+10:30  — Real data owner shard assignments begin
```

**3. `scripts/test/demo_timeline_test.go` (Session 16.1.1) is re-sequenced to match.** `uploadTestFile` moves to after `pollAllProvidersActive` succeeds, not before — confirming providers are `ACTIVE` before attempting the upload that the corrected timeline says should succeed. This is a sequencing fix to an already-written test, not new code; every helper function the corrected order needs (`pollAllProvidersActive`) already exists in the file.

**4. No change to IC §5.9.** Its documented `ErrNetworkNotReady`/503 retry semantics already correctly describe the T+01:00 rejection this ADR's corrected timeline now shows explicitly.

## Consequences

The demo timeline becomes internally consistent and consistent with ADR-030 — a live audience sees an honest sequence: registration and escrow succeed immediately, the first upload attempt is visibly rejected while vetting is in progress (not silently skipped), and the real upload succeeds once providers are genuinely trustworthy. This is a more accurate demonstration of the product's actual trust model, not a weaker one.

Cost: one `mvp.md` edit, one test re-sequencing. No production code touched, no risk to ADR-030's ratified trust boundary.

**Open constraints:** none introduced by this ADR. The demo's own retry-and-wait pacing during T+01:00–T+10:00 is a presentation concern (what a live presenter says while waiting), not an engineering one, and is out of this ADR's scope.

---
