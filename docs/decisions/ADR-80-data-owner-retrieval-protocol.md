# ADR-080 — Data Owner Retrieval: Authorized Chunk Download

**Status:** Accepted
**Track:** Both — the demo track needs this to retrieve a file at all; the design is the intended LTS form, not a demo shim
**Topic:** #4 Wire Protocols / #10 Key Management Strategy
**Supersedes:** — *(ratifies, with substantial modification, the two PROPOSED-NOT-RATIFIED
additions flagged in `internal/client/retrieve/download.go`'s header at M15)*
**Research source:** Design Council verdict "Data Owner File Retrieval" (M17 Session 17.2.1
live verification); direct audit of `internal/api/upload.go`, `cmd/provider/handler_upload.go`,
`cmd/provider/handler_repair.go`, `internal/client/retrieve/{download,pointer}.go`,
`migrations/001_initial_schema.sql`, IC §4.1/§4.2/§4.4.1/§4.5

---

## Context

`internal/client/retrieve` has, since M15, implemented a complete client-side download path
against two endpoints that **do not exist**. Verified by direct audit, not inference:

- `grep -rn "providers/resolve" internal/api/*.go` → **0 matches**.
- `grep -rn "chunk-download" cmd/provider/*.go internal/p2p/*.go` → **0 matches**; the
  provider's own startup log lists exactly four registered stream handlers, none of them a
  chunk-download protocol.
- `scripts/test/demo_timeline_test.go` (M16) never calls `RetrieveFile`.

The consequence, stated plainly: **no data owner has ever successfully retrieved a file from
this system.** M17 Session 17.2.1's `TestDemoCLIFullLifecycle` is the first test that ever
exercised the path, and it failed at `resolveProviderAddresses: unexpected status 404`.

M15's author saw this and flagged it accurately in `download.go`'s header as a "genuine
architecture gap, not an implementation detail," implementing two placeholder proposals
explicitly "so this session has something concrete and testable to build against," while
recording that the security-relevant half "belongs in front of the design council, not decided
unilaterally here." This ADR is that council decision.

### Why the gap existed

IC §4's protocol family encodes an unstated assumption: **the microservice is the only entity
that reads shard data.** Upload (§4.1) is owner→provider *write*. Repair download (§4.4.1) is
microservice→provider *read*. Audit (§4.2) returns a proof, never data. Vetting GC (§4.5) is a
delete instruction. Every read path is microservice-initiated — which is why §4.4.1's
authentication is literally "is the caller a registered microservice replica," rejecting
everyone else with `0x02 NOT_AUTHORISED`. A data owner's Peer ID is not, and must not be, on
that list.

FR-016 and IC §5.9's `RetrieveFile` are both real, ratified requirements. So this is not an
oversight in a complete taxonomy — it is a **fourth participant role** (owner-as-reader) added
to a protocol family designed around two.

## The rejected alternative, and why

M15's placeholder proposed a bare `chunk_id` request with no authentication, on the model that
"knowledge of the hash is the capability" — as several content-addressed P2P systems do.
Rejected on three independent grounds:

1. **It breaks `rm`.** After `DeleteFile` marks assignments `PENDING_DELETION`, anyone holding
   the old `chunk_id`s can keep fetching shards until physical GC completes — which, for an
   offline provider, is unbounded (§4.5 retries on next heartbeat). The delete guarantee
   becomes unenforceable. This is the same class of problem ADR-036 closed for *mutations*;
   reads were simply never considered.
2. **It conflates integrity with authorization.** `chunk_id` is `SHA-256(chunk_data)` (IC §4.1,
   ADR-073) and is the *integrity* mechanism, verified client-side after every fetch. It is
   stable for the file's entire lifetime, identical across all holders, and known to every one
   of the 56 daemons holding a shard plus every `upload/assign` response. An identifier with
   those properties is a fine content address and a terrible secret. Authorization needs a
   value that is scoped, expiring, and revocable; integrity needs one that is stable and
   public. **They must not be the same value.**
3. **Its usual justification does not apply here.** Hash-as-capability is normally chosen to
   avoid a coordinator on the read path. But `internal/client/retrieve/pointer.go` **already**
   fetches the encrypted pointer file from the microservice over REST before any shard is
   dialled. A retrieval with a dead microservice already fails at step one. There is therefore
   no availability argument for it — only a complexity argument, and the complexity in question
   is already written, shipped, and tested for the write direction.

AONT-RS means a single shard is not plaintext (k=16 prod, k=3 demo), so this would not have
been an immediate confidentiality break. It would have been a permanent, unrevocable erosion of
defense-in-depth for no benefit this system actually collects.

## Decision

### 1. Reuse the capability-token model, with strict domain separation

IC §4.1's upload `capability_token` is already implemented **on both sides** and verified in
production code: a 72-byte token, domain-separated Ed25519 by the microservice signing key over
`chunk_id ‖ provider_id ‖ expiry_unix_ms`, checked in `cmd/provider/handler_upload.go` with a
30-second clock-skew grace. The daemon already holds `msPublicKey`.

Download is that same sentence with one word changed: *the microservice vouches that this
client may **read** this chunk from this provider, until this timestamp.*

The download token is therefore structurally identical but uses a **distinct domain-separation
prefix**, so a token minted for one direction can never be replayed as the other:

```
upload   prefix: "vyomanaut-chunk-upload-cap-v1"     (existing, unchanged)
download prefix: "vyomanaut-chunk-download-cap-v1"   (new)

download signing_input = prefix ‖ chunk_id(32) ‖ provider_id(16) ‖ expiry_unix_ms(8 be)
download cap_sig       = Ed25519(microservice_signing_key, signing_input)   → 64 B
```

**Every signed field is transmitted in Frame 1.** This is deliberate and non-negotiable: IC
§4.4.1 shipped a signing formula over `request_ts_ms`, a field its own Frame 1 did not carry,
and the responder consequently could not verify it — the REPAIR-AUTH-TS-GAP finding, corrected
in `handler_repair.go` by extending that frame to 104 bytes. This ADR does not repeat that
mistake.

This introduces **no new key material and no new trust root**: the provider verifies with the
same `msPublicKey` it already uses for upload tokens.

### 2. `POST /api/v1/owner/files/{file_id}/retrieve/resolve`

Owner-authenticated (JWT `sub` must equal `files.owner_id`); the file must be `ACTIVE`.
Returns, for **every segment of the file in a single call**, each shard's `provider_id`,
`multiaddrs`, `multiaddr_stale`, and its download capability token.

- **Batching is a requirement, not an optimization.** `downloadSegment` currently resolves
  per-segment. At prod parameters a 1 GB file is ~1,365 segments; per-segment resolution is
  ~1,365 REST round-trips before a single byte of data moves. One call, keyed by segment index,
  is the correct shape — the client already holds the entire decrypted pointer file (and thus
  every `provider_id`) before it starts.
- **No new disclosure category.** `POST /api/v1/upload/assign` already returns
  `multiaddrs` to data-owner clients (`internal/api/upload.go`, joined from
  `providers.last_known_multiaddrs`). Exposing provider addresses to an authenticated owner for
  chunks that owner owns is already ratified practice; this endpoint is symmetric with it.
- `multiaddr_stale` (present in the schema since migration 001) is surfaced so the client can
  prefer fresh addresses rather than dialling a known-dead one.

Revocation is by **non-issuance**: the moment a file leaves `ACTIVE`, the microservice stops
minting tokens for its chunks, and outstanding tokens expire on their own short clock. This is
what restores the `rm` guarantee that hash-as-capability would have broken.

### 3. `/vyomanaut/chunk-download/1.0.0`

```
Frame 1 — ChunkDownloadRequest
| length         | uint32 be | 4 B  | Must equal 32 + 8 + 64 = 104
| chunk_id       | bytes     | 32 B | Content address of the requested shard
| expiry_unix_ms | int64 be  | 8 B  | Token expiry; signed AND transmitted
| cap_sig        | bytes     | 64 B | Ed25519 over the §1 signing_input

Frame 2 — ChunkDownloadResponse   (mirrors §4.4.1 exactly)
| length     | uint32 be | 4 B      | Success: 1 + 262144. Error: 1
| status     | uint8     | 1 B      | 0x00=OK 0x01=NOT_FOUND 0x02=NOT_AUTHORISED
|            |           |          | 0x03=CORRUPTION 0x04=INTERNAL_ERROR
| chunk_data | bytes     | 262144 B | Present only when status = 0x00
```

**0-RTT: PROHIBITED**, for §4.4.1's reason exactly — a replayed stream could exfiltrate chunk
data to an unauthenticated party.

**Timeout: 10,000 ms**, matching §4.4.1 rather than the shorter upload timeout, for the same
cold-disk-read justification.

**Integrity is still verified client-side.** `fetchOneShard` already checks
`SHA-256(data) == chunk_id` before handing anything to RS decode, and that check stays. The
token authorizes; the content address verifies. Separate values, separate jobs (see Context §2).

### 4. Status-code semantics are security-relevant, not mechanical

A naive mirroring of §4.4.1 leaks holder-set membership: an unauthenticated prober with a
guessed `chunk_id` could distinguish "this provider holds it" from "it does not" by the error
code alone. Therefore:

- **Token fails to verify (bad signature, expired, wrong provider) → `0x02 NOT_AUTHORISED`,
  always** — regardless of whether the chunk is present. The prober learns nothing.
- **Token verifies but the chunk is genuinely absent → `0x01 NOT_FOUND`.** This leaks nothing
  new: a valid token is itself proof that the microservice told this caller that this provider
  holds this chunk. Debuggability is preserved exactly where it is safe.

### 5. No enumeration, ever

The protocol has exactly one request shape: a single named `chunk_id` with a matching token.
There is no listing operation, no wildcard, no range request, and none may be added without a
superseding ADR. A provider must never expose "what do you hold" to any caller on this path.

### 6. IC §4 gains an explicit statement of participant roles

IC §4's preamble must state that **four** participant roles now exist, and that owner-initiated
*reads* are a first-class protocol category — not an exception. The microservice-is-the-only-
reader assumption was never written down, which is precisely why it survived long enough to
produce this gap. Left implicit, the next protocol author inherits it.

## Consequences

**Positive:** retrieval works at all, for the first time. The authorization model is the one
already built, tested, and ratified for upload — no new cryptographic machinery, no new trust
root, one added domain-separation constant. `rm`'s delete guarantee becomes enforceable on the
read path. Batched resolution removes ~1,365 serial round-trips from a 1 GB retrieval.

**Negative / trade-offs:** the microservice is explicitly on the retrieval critical path. This
is a real coupling, but not a *new* one (pointer-file fetch already required it), and it is
what makes revocation possible at all. Token issuance adds per-retrieval signing work
proportional to shard count. `internal/client/retrieve/download.go`'s Frame 1 and its
resolution call both change, so M15's already-written client code moves rather than merely
being connected.

**Affected:** `internal/api/` (new resolve handler + route), `cmd/provider/` (new
chunk-download stream handler), `internal/client/retrieve/download.go` (Frame 1 gains the
token; per-segment resolution becomes whole-file), `docs/api/openapi.yaml`,
`interface-contracts.md` (new §4.6 + §4 preamble amendment per Decision §6).

**Open constraints:**

- **Q-ADR80-1:** should download tokens be scoped per `(chunk_id, provider)` as decided here,
  or per `(segment, provider)` to cut token volume on very large files? At prod parameters a
  1 GB file needs ~76,000 tokens (~5.5 MB) if all are issued eagerly. Batched-but-lazy issuance
  (resolve returns tokens for a bounded window of segments, client re-calls as it advances)
  is the likely answer and does not change the wire protocol — deferred, not blocking, since
  demo-track files are far below the threshold where this matters.
- **Q-ADR80-2:** token expiry duration is not fixed by this ADR. It trades revocation latency
  against retrying a slow download. It belongs in `NetworkProfile` (demo/prod split), not as a
  hardcoded constant, and inherits the `[UNDERIVED]` governance of ADR-077 until derived.
- **Q-ADR80-3:** should a provider rate-limit chunk-download requests per requesting Peer ID?
  Not required for correctness, and out of scope here, but a valid token does not bound request
  *volume* — relevant to LTS abuse-resistance work.

## References

- Design Council verdict "Data Owner File Retrieval" (M17 Session 17.2.1) — the five-seat
  session this decision comes from
- IC §4.1 — the upload `capability_token` this design mirrors; §4.4.1 — repair download, the
  frame/status template and the REPAIR-AUTH-TS-GAP precedent this ADR deliberately avoids
  repeating; §4.2, §4.5 — the other protocols, confirming the microservice-only-reader pattern
- [ADR-036](./ADR-036-authenticated-provider-mutation-protocols.md) — authenticated provider
  mutation protocols; this ADR is its read-path counterpart
- [ADR-073](./ADR-073-initial-upload-assignment-client-chunk-id.md) — establishes
  `chunk_id = SHA-256(chunk_data)`, the property that makes it unsuitable as an authorization
  secret
- [ADR-077](./ADR-077-research-first-triage.md) — `[UNDERIVED]` parameter governance, which
  Q-ADR80-2 inherits
- FR-016, IC §5.9 `RetrieveFile` — the ratified requirements this gap was blocking
- `internal/client/retrieve/download.go` header (M15) — the original flag; its analysis of the
  gap is confirmed correct, its proposed hash-as-capability resolution is rejected above
