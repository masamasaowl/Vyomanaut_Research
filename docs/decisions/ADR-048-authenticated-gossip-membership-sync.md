# ADR-048 — Authenticated, Freshness-Bound Inter-Replica Gossip Membership Sync

**Status:** Accepted
**Topic:** #1 Coordination Architecture, Security
**Supersedes:** —
**Superseded by:** —
**Research source:** interface-contracts.md §2 (Component Communication Map and its own
undiagrammed-link rule), §8 (Secrets Manager Contract — path/rotation precedent), §12
(DHT Key Contract — precedent for a locally-cached, refreshed peer-authorization set);
architecture.md §18 (Coordination Microservice — gossip membership, quorum, client-driven
routing); ADR-025 (microservice-consistency-mechanism — the quorum arithmetic this
protocol carries); ADR-036 (authenticated-provider-mutation-protocols — the
authenticated/freshness-bound pattern this ADR extends one layer up the stack);
`Vyomanaut_V2` @ current `main`, `build_part3.md` Milestone 17 Session 17.2.1

---

## Context

`interface-contracts.md` §2's Component Communication Map is, by its own governing
sentence, *"the single authoritative picture of which components are allowed to talk to
which. A code path that creates a communication link not shown here is out of scope for
V2 and requires a new ADR before implementation."* The diagram has exactly one `MS` node,
annotated *"3 replicas, gossip cluster,"* and no edge from `MS` back to itself. No edge,
no cross-reference-table row, no ADR.

`architecture.md` §18 nonetheless already commits to the traffic that edge would carry:
*"each replica contacts one randomly chosen peer per second to reconcile membership
histories."* `build_part3.md`'s Milestone 17 Session 17.2.1 turns that sentence into a
concrete wire contract: `internal/cluster/gossip.go`'s `reconcile()` loop exchanges
membership histories with one randomly-chosen peer per second via `POST
/internal/membership/sync`, carrying a vector clock, with **no authentication field of
any kind** in the request as specified. `HealthyCount()` — the function `ADR-025`'s
`(3,2,2)` quorum and the readiness gate both depend on — is a pure function of what this
endpoint receives.

This is not a hypothetical gap. `ADR-036` (Accepted-pending, this same review cycle)
already established, for a structurally identical problem one layer down the stack, that
an unauthenticated mutating endpoint on a security-critical path is a real vulnerability,
not a theoretical one: `/vyomanaut/vetting-gc/1.0.0` could be pointed at arbitrary real
chunk IDs by anyone able to open the stream, because *"chunk IDs are DHT-discoverable...
and the provider daemon cannot distinguish synthetic from real"* (DM §3 Invariant 6). The
gossip-sync endpoint has the same shape of exposure for a different asset: `HealthyCount()`
and `ResponsibleReplica()` drive which replica is treated as authoritative for
audit-challenge dispatch and chunk-assignment decisions (architecture.md §18,
"client-driven routing"). A forged or replayed sync payload that inflates or deflates a
replica's apparent health does not delete data the way a forged GC instruction does, but
it can manufacture a false quorum, or falsely evict a healthy replica from the routing
table, on a control plane that already handles payment-consequential decisions (release
computation, escrow seizure) elsewhere in the same process. `ADR-036`'s own closing
argument — *"the only remaining defense... is absent"* — applies here verbatim; the
1-second gossip cadence and the (3,2,2) quorum arithmetic are both silent on who is
allowed to claim membership on a peer's behalf.

## Decision

**1. Close the diagram gap.** Add an `MS -- "HTTPS, mutually authenticated (this ADR)\n
(gossip membership sync)" --> MS` self-edge to `interface-contracts.md` §2's Mermaid
diagram, and a corresponding row to the cross-reference table pointing at this ADR. This
satisfies §2's own precondition for the link Session 17.2.1 already builds.

**2. Each replica gets its own Ed25519 identity, distinct from the shared audit secret.**
The cluster audit secret (`ADR-027`) is a symmetric HMAC key shared identically across all
three replicas by design — it proves "some replica knows the secret," never *which* one,
and reusing it here would couple two unrelated security domains: a future audit-secret
rotation incident would also become a cluster-membership incident. Instead, each replica
generates an Ed25519 key pair at first startup, persisted locally in the same pattern as a
provider daemon's own identity (`internal/p2p/identity.go`, architecture.md §13). This
mirrors why providers already have per-node Ed25519 identities rather than a shared
network-wide secret — the same reasoning applies one tier up.

**3. Public keys are distributed through the existing secrets-manager contract, at a new
path.** `interface-contracts.md` §8's `SecretsManagerClient.GetSecret(ctx, path)` interface
is unchanged — no new interface, only a new path convention alongside the existing
`/vyomanaut/audit-secret/v{N}`:

```bash
/vyomanaut/cluster-replica-pubkeys/v{N}
```

Value shape: a small JSON object, `{"replica_id": "<32-byte hex Ed25519 public key>", ...}`
for the current three replicas — the one deliberate divergence from the audit-secret
path's raw-32-byte shape, since this path always holds a small *set* of keys rather than
one rotating scalar. Each replica reads and caches this path on the same 5-minute TTL
already governing the audit secret (§8), reusing the identical fail-closed-at-startup /
serve-cached-for-5-minutes-then-`ErrSecretExpired` behavior — no new caching policy to
design or review.

**Rotation** does not follow the audit secret's automatic 24-hour overlap window, because
replica identity changes only on replica replacement, not on a schedule. Rotation is
operator-driven: replacing a replica generates a new key pair and updates this path; the
two surviving replicas pick up the change on their next 5-minute cache refresh. This is
recorded as a new step in `runbooks/microservice-failover.md` (already required to exist
by Milestone 18 Session 18.1.1), not as a new automated mechanism.

**4. Wire format — both directions of the exchange are authenticated and freshness-bound,**
extending `MembershipSyncRequest`/`-Response` with the same two fields `ADR-036` already
introduced for provider-mutation protocols, reusing rather than re-deriving the profile
field:

| Field | Type | Size | Description |
| --- | --- | --- | --- |
| `replica_id` | UUID bytes | 16 B | Sender's replica identity (existing field) |
| `vector_clock` | bytes | variable | Existing membership-history payload (unchanged) |
| `request_ts_ms` | int64 be | 8 B | **New** — freshness anchor |
| `replica_sig` | bytes | 64 B | **New** — Ed25519 by the sender's own replica key over `SHA-256("vyomanaut-gossip-sync-v1" ‖ replica_id ‖ vector_clock ‖ request_ts_ms)` |

The response carries the identical two new fields, signed by the responder's own replica
key over the same domain-separated construction — this is a bidirectional reconciliation,
and a forged *response* can poison a requester's membership view exactly as effectively as
a forged request.

**5. Handler ordering**, matching the authz-before-mutation pattern `ADR-036` already
established and its own `VERIFY` checks enforce elsewhere in this build: on receipt, a
replica MUST, in order, before merging anything into local membership state: (a) verify
`replica_id` is one of the two other keys in the cached `cluster-replica-pubkeys` set —
else discard silently (an unrecognized replica ID is not a protocol error worth a response,
just a message that does not get merged); (b) verify `|now − request_ts_ms| ≤
profile.AuthRequestFreshnessWindow` (the same field `ADR-036` introduced — no second
freshness constant) — else discard as stale; (c) verify `replica_sig` — else discard.
Only a payload passing all three is merged into the local vector clock.

**6. `HealthyCount()`'s liveness window becomes a named profile field.** The current
`build_part3.md` text hardcodes *"peers with a last-seen timestamp within the last 5
seconds"* as a bare literal — a magic number, in a project whose own Forbidden Patterns
section treats that as a defect class in everything else it specifies. This ADR promotes
it to `NetworkProfile.GossipHealthyWindow`:

| Field | Production | Demo |
| --- | --- | --- |
| `GossipHealthyWindow` | `5s` | `5s` |

Both modes carry the same value — demo mode runs a single instance with
`RequireQuorum=false` (mvp.md §3.1), so `HealthyCount()` never gates anything there; the
field still needs a value in both profile instances per `mvp.md` §5.1's single-struct
principle and `TestProfileBothFullySpecified`. The value itself is 5× the 1-second gossip
cadence architecture.md §18 already specifies — tolerating up to four consecutive missed
reconciliation cycles before a peer drops out of the healthy count — recorded here as an
explicit rationale rather than an unexplained constant. Like `ADR-036`'s freshness window
and `ADR-025`'s background-throttle threshold, this is a starting value, not empirically
tuned; it is a `NetworkProfile` field specifically so it can be adjusted without a code
change.

## Alternatives considered

- **Reuse the shared cluster audit secret (`server_secret_vN`, ADR-027) as an HMAC key for
  gossip sync, instead of new per-replica Ed25519 identities.** Rejected: it authenticates
  "a member of the cluster," never *which* member — a compromised or misbehaving replica
  could forge a `vector_clock` entry claiming to speak for a peer, indistinguishable from
  that peer's genuine traffic — and it couples two independent security domains (audit
  validation, cluster membership) behind one secret, so a future incident affecting one
  compromises the other for free. Every other authenticated mutation in this system (chunk
  upload receipts, audit responses, ADR-036's two protocols) already uses per-identity
  Ed25519 signatures for exactly this reason; gossip sync is not a principled exception.
- **Rely on network segmentation (firewalled internal-only port) with no application-layer
  authentication.** Rejected for the same reason `ADR-036` rejected the equivalent
  alternative for provider protocols: it authenticates network position, not the payload —
  a compromised host anywhere on the internal segment, or a misconfigured firewall rule
  during the "Production Hardening" milestone this ADR belongs to, produces a silent gap
  with no defense-in-depth behind it. Network segmentation remains good practice
  *alongside* this decision, not instead of it.
- **mTLS between the three replicas instead of per-message application-layer signatures.**
  Considered. Would authenticate the channel, and is a reasonable design in the abstract.
  Rejected here because (a) it is inconsistent with how every other mutating protocol in
  this system is secured — payload-level Ed25519 signatures over a fixed, domain-separated
  byte layout, independent of which connection carried them (the same reasoning ADR-036
  used to reject channel-identity-only authorization) — and mixing a transport-level
  scheme into one specific link would be a second authentication pattern to maintain for no
  corresponding benefit; and (b) standing up a private CA and certificate lifecycle for a
  fixed set of exactly three, rarely-changing peers is disproportionate operational
  machinery next to a three-entry public-key list served through infrastructure (the
  secrets manager) this system already operates for a near-identical purpose.
- **Fold this into ADR-036 as a third protocol.** Rejected, matching ADR-036's own
  reasoning for not folding *its* two protocols into ADR-030: ADR-036 is scoped, by its own
  text, to provider-daemon mutation protocols reached from the microservice; this decision
  secures a structurally different link — microservice replica to microservice replica —
  introducing new key material (replica identities) and a new secrets-manager path that
  ADR-036 has no reason to own. A future reader looking for "how is inter-replica gossip
  secured" should find it under its own title, not as a subsection of a provider-protocol
  ADR.

## Consequences

**Positive:** closes the undiagrammed-link gap `interface-contracts.md` §2 itself flags as
blocking; brings the one remaining unauthenticated mutating-in-effect endpoint in the V2
build plan up to the same authenticated/freshness-bound standard every other
security-relevant protocol in this system already meets (chunk upload capability tokens,
audit challenge/response, and — pending — vetting-GC/repair-download via ADR-036); reuses
existing infrastructure on every axis (the `SecretsManagerClient` interface, the 5-minute
cache TTL pattern, `AuthRequestFreshnessWindow`) rather than introducing new machinery;
gives `microservice-failover.md` a concrete, ADR-grounded step it was otherwise missing.

**Negative / trade-offs:** wire-format change to `MembershipSyncRequest`/`-Response` before
this protocol has any production traffic (cheapest possible time to make it, same framing
ADR-036 used); adds one new piece of key material (three small Ed25519 key pairs, one per
replica) and one new secrets-manager path to operate, on top of the existing audit-secret
path; a signing step on every 1-second gossip tick on both sides of the exchange — bounded,
inexpensive (Ed25519 sign/verify is sub-millisecond at this message size), but not zero
cost on the hottest-cadence link in the system.

**Open constraints:**

- `GossipHealthyWindow = 5s` is a starting value, consistent with this project's own
  precedent of not pretending a first estimate is a tuned one (`ADR-025`'s 50 ms background
  throttle threshold, `ADR-036`'s 120 s freshness window) — subject to revision after
  observing real replica behavior under the production (3,2,2) topology.
- This ADR does not address what happens if a replica's local private key is lost (e.g.
  disk failure) — the natural answer is "treat it as a replica replacement, generate a new
  key pair, update the secrets-manager path," i.e. the same runbook path as routine
  rotation, but this is noted here as unelaborated rather than silently assumed.

## Affected

`interface-contracts.md` §2 (diagram + cross-reference table), §8 (new path convention,
same interface); `architecture.md` §18 (cross-reference to this ADR alongside the existing
gossip-membership paragraph); `internal/config` (two new `NetworkProfile` fields:
`GossipHealthyWindow`; reuses existing `AuthRequestFreshnessWindow`); `internal/cluster`
(new — `gossip.go`'s wire format, key generation/loading, verification ordering);
`runbooks/microservice-failover.md` (new rotation step); `build_part3.md` Session 17.2.1.

## References

- [ADR-025](./ADR-025-microservice-consistency-mechanism.md) — the (3,2,2) quorum and
  gossip-membership decision this protocol carries; unaffected by this ADR
- [ADR-027](./ADR-027-cluster-audit-secret.md) — the secrets-manager contract and
  5-minute-cache pattern this ADR reuses for a new path; the shared-HMAC design this ADR
  deliberately does not reuse for replica identity
- [ADR-036](./ADR-036-authenticated-provider-mutation-protocols.md) — the
  authenticated/freshness-bound pattern this ADR extends one layer up the stack, and the
  origin of `AuthRequestFreshnessWindow`, reused rather than re-derived here
- `interface-contracts.md` §2 — the Component Communication Map and its own
  undiagrammed-link-requires-an-ADR rule, which this ADR satisfies
- `architecture.md` §13 — per-provider-daemon Ed25519 identity at installation, the
  existing precedent this ADR mirrors for per-replica identity
