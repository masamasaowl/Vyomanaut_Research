# ADR-036 — Authenticated, Freshness-Bound Provider-Mutation Protocols

**Status:** Accepted
**Topic:** #1 Coordination Architecture, #5 Peer Selection, Security
**Supersedes:** ADR-030 §"New libp2p sub-protocol for vetting GC" — specifically its authorization posture only. ADR-030's vetting mechanism (synthetic chunks, GC timing, ACTIVE-transition flow) is unchanged.
**Superseded by:** —
**Research source:** IC §4.4.1 (repair-download auth precedent), IC §4.5, DM §3 Invariant 6, ADR-021 (0-RTT disabled for mutating protocols), ADR-030 (synthetic vetting chunks); `Vyomanaut_V2` architecture/build review, Milestones M-OBS/13/14

---

## Context

Two provider-daemon protocols cause **irreversible or exfiltrating side effects**:

- `/vyomanaut/vetting-gc/1.0.0` — permanently **deletes** chunks (`DeleteChunk`).
- `/vyomanaut/repair-download/1.0.0` — **returns raw shard bytes**.

Repair-download is authenticated (registered-microservice Peer ID + `repair_auth_sig`) but its signed `request_ts_ms` is **never checked for freshness** by the handler, so a leaked signature is a permanent bearer credential for that chunk. Vetting-GC has **no authorization field or check whatsoever**.

**Why vetting-GC's gap is real and not just theoretical.** ADR-030 designed the vetting-GC protocol on the reasoning that *"a vetting GC stream will never carry real chunk IDs since real chunks are never assigned to vetting providers"* — treating the 0-RTT prohibition as a defensive-only measure rather than the load-bearing control. That reasoning holds for a *legitimate* instruction from the real microservice. It does not hold for a *forged* one: chunk IDs are DHT-discoverable (one provider record per chunk, published for lookup), and `DM §3 Invariant 6` states the provider daemon *"cannot distinguish synthetic from real"* — by design, so vetting chunks are indistinguishable from production data during audit. The daemon therefore has no content-based defense against a forged instruction, and the only remaining defense — caller authorization — is absent. Any peer able to open the protocol stream can instruct permanent deletion of arbitrary real chunks, forcing repairs and degrading the 10⁻¹⁵ durability guarantee. IC §4.5's replay note (idempotent re-delete) protects against replaying a *previously legitimate* instruction; it says nothing about a *forged* one.

Both problems are cheapest to fix now: the network is pre-GA, so this is a same-version correction rather than a coordinated live migration.

## Decision

**1. Vetting-GC gains authorization at parity with repair-download.** Extend `VettingGCRequest` Frame 1:

| Field | Type | Size | Description |
| --- | --- | --- | --- |
| `length` | uint32 be | 4 B | `4 + (chunk_count×32) + 8 + 64` |
| `chunk_count` | uint32 be | 4 B | ≤ 10 000 |
| `chunk_ids` | bytes | `chunk_count×32` | targets |
| `request_ts_ms` | int64 be | 8 B | **new** — freshness anchor |
| `gc_auth_sig` | bytes | 64 B | **new** — Ed25519 by the microservice signing key over `SHA-256(chunk_ids ‖ request_ts_ms ‖ microservice_peer_id)` |

The provider handler MUST, in order: (a) verify the requesting Peer ID ∈ locally-cached registered-microservice set → else `0x03 NOT_AUTHORISED`; (b) verify `|now − request_ts_ms| ≤ AuthRequestFreshnessWindow` → else `0x04 STALE_REQUEST`; (c) verify `gc_auth_sig` → else `0x03`; only then delete. Add `0x03 NOT_AUTHORISED` / `0x04 STALE_REQUEST` to `VettingGCResponse` (renumber the existing `INTERNAL_ERROR` to `0x05`).

**2. Repair-download gains the freshness check it already signs for.** The §4.4.1 handler MUST reject when `|now − request_ts_ms| > AuthRequestFreshnessWindow` with `0x02 NOT_AUTHORISED` (no new status needed; the field is already signed).

**3. New profile field.** `AuthRequestFreshnessWindow` — prod `120 s`, demo `120 s` (generous enough for clock skew + relay latency; small enough to bound replay). It is a `NetworkProfile` field, never hardcoded.

## Alternatives considered

- **Rely on the 0-RTT prohibition alone, as ADR-030 originally reasoned.** Rejected: this defends against replay of captured traffic, not against a forged instruction from an unauthenticated caller — a different threat entirely, and the one actually exploitable here.
- **Fold this fix into ADR-030 as an amendment.** Considered, but rejected in favor of a standalone ADR: the decision establishes a *general* authenticated-mutation pattern applying to two protocols from two different origins (vetting-GC, owned by ADR-030; repair-download, specified directly in IC §4.4.1 with no owning ADR at all), plus a new shared `NetworkProfile` field intended to cover future mutating protocols too. Filing the repair-download half of the fix under an ADR titled "Synthetic Vetting Chunks" would misdirect a future reader. ADR-030's vetting-chunk *design* is otherwise sound and unchanged; only its authorization posture is superseded here.
- **Bind authorization to TLS/QUIC channel identity alone, no application-layer signature.** Rejected (implicit in the original proposal): channel identity confirms the transport peer, not that the *payload* (specific chunk IDs, specific timestamp) was authorized by the microservice's signing key — the signature is what makes the instruction itself non-forgeable, independent of which connection carried it.

## Consequences

**Positive:** closes the unauthenticated-deletion hole; bounds replay of both mutating protocols; brings the two side-effecting protocols to a single, testable authorization pattern that future mutating protocols can reuse.

**Negative / trade-offs:** wire-format change to two versioned protocols (must land before GA, or require coordinated migration after — IC §13); adds a signing step to the GC client (Session 14.2.1) and clock-skew sensitivity (mitigated by the 120 s window).

**Affected:** IC §4.4.1, IC §4.5; `internal/config` (new field); Sessions 13.4.1, 13.5.1, 14.2.1.

## Renumbering note

This proposal referenced itself as "ADR-032" in the source review because ADR-032/033 did not yet exist at the time it was written. The real `ADR-032-rls-role-model.md` and `ADR-033-audit-receipts-partitioning.md` have since been accepted, so this file is issued as **ADR-036** instead. Every "apply after ADR-032 accepted" marker in the M-OBS/13/14 review's rewritten sessions (Sessions 13.4.1, 13.5.1, 14.2.1; the sign-off checklist; the `NFR-040` CI-gate comment) needs its placeholder updated to **ADR-036** before that review is used to drive a build session — flagging this explicitly rather than silently fixing it, since it touches ~15 locations across a document you may want to re-diff yourself.
