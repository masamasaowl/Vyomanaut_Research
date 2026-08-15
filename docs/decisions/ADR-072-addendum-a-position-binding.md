# ADR-072 Addendum A — The Capability Token Binds the Assignment, Not the Payload

**Status:** Accepted
**Track:** LTS
**Topic:** #14 Client Interface / #13 Provider Daemon Core
**Amends:** ADR-072 (whose *decision* stands unchanged; its *justification* is replaced)
**Research source:** Design Council session 5, August 2026; ADR-073's correction of the inherited
`chunk_id` assumption

---

## Why this addendum exists

ADR-072 removed `file_id` from the `capability_token` signing input. The decision was correct and
remains in force. The argument recorded for it was:

> `chunk_id` is 256 bits of fresh, microservice-generated randomness, minted exactly once per
> assignment … and never reused across files.

ADR-073 established one session later that `chunk_id` is exactly `SHA-256(chunk_data)` per IC §4.1 —
a **content address, computed and submitted by the client**, which the microservice cannot verify
because it never sees `chunk_data`. Every clause of the quoted argument is false of that object: it
is not fresh randomness, not microservice-generated, and not guaranteed unique across files by
construction.

ADR-073 correctly declined to reopen ADR-072's decision. This addendum closes the remaining gap:
**the decision is now correct for a reason that is true.**

## The residual exposures ADR-072's argument was covering for

| # | Exposure | Reachability |
| --- | --- | --- |
| **1** | **Cross-file token transfer.** If two files ever yield the same `chunk_id` at one provider, a token minted for one authorises a write for the other | Requires an AONT key collision — F-47's RNG-failure scenario (VM cloning, image reuse, snapshot restore, early-boot entropy starvation). Low probability, silent, and it converts an RNG bug into an authorisation bypass |
| **2** | **Address squatting.** An attacker uploads a file, observes its own `chunk_id`s, then submits one in a *second* file's assignment request. The provider's `0x02 CHUNK_ID_MISMATCH` check passes, because the attacker possesses the matching bytes | Reachable today. Two segments in two files name one stored object: one copy billed twice, and either owner's `rm` destroys the other's shard. A billing and durability break, not a confidentiality one |

Neither is closed by the current signing input, because that input names *what bytes to write*
rather than *which assignment is being fulfilled*.

## Decision

### 1. Bind position into the signing input

IC §4.1's `capability_token` signing input becomes:

```
domain_prefix || chunk_id(32) || provider_id(16) || segment_id(16) || shard_index(4) || expiry_unix_ms(8)
```

**Zero wire cost.** The signing input is never transmitted — both parties construct it
independently. `shard_index` is already carried in Frame 1. `segment_id` is available to the client
from the assignment response and to the provider from the `chunk_assignments` row it already reads
to obtain `provider_id`. This is the fix ADR-073's Option (C) was reaching for, and it does not hit
the wall that killed Option (C): Frame 1's fixed 262,252-byte layout is **unchanged**.

The token now names the assignment again. Exposure 1 closes: a token bound to
`(segment_id, shard_index)` is not transferable to a different file's segment even under a
`chunk_id` collision. **F-LTS-01 closes**, and authorisation integrity stops depending on AONT key
freshness — Domain N returns to being about confidentiality only.

### 2. Prohibit duplicate `chunk_id` per provider

Exposure 2 is a **specification gap**, not a token gap. `data-model.md` gains a constraint: two
`chunk_assignments` rows for one provider must not name the same `chunk_id`, enforced by a partial
unique index following the existing `idx_chunk_assignments_one_active_per_shard` precedent.
**F-LTS-02 closes.**

### 3. Record the deliberate trade

Vyomanaut uses content addressing for integrity and pays its costs — a client-supplied identifier
the server cannot verify, a global namespace, squatting exposure — and **deliberately forgoes
deduplication**, because cross-owner dedup leaks content equality. Stated here so a future
contributor does not "fix" the missing dedup.

### 4. State the invariant that is currently held by accident

`chunk_id` is SHA-256 over AONT-RS ciphertext encrypted with a **fresh per-segment random key**, so
identical plaintexts yield different `chunk_id`s and the convergent-encryption attack family does
not apply. That property follows from ADR-019, not from anything ADR-073 says, and it is
load-bearing. It becomes a stated invariant. **R-60 closes by construction on this basis**, pending
one verification: confirm the client generates a fresh AONT key per segment *per upload*, including
on re-upload of an identical file.

### 5. One sentence in IC §4.1

*The microservice does not and cannot validate `chunk_id`; the storing provider is the enforcement
point.* ADR-072 was misled by a stale M13-era code comment on exactly this. The remedy is to write
it down.

## Implementation

Same four sites ADR-072 already touched — `internal/api/upload.go` (`respondWithFreshTokens`),
`internal/repair/executor.go` (`mintCapabilityToken`), `internal/vettingchunk/generator.go`, and
`cmd/provider/handler_upload.go` (`capabilityTokenSigningInput` / `verifyCapabilityTokenFrame`).
No wire change, no OAS change, no migration. Roughly half a session.

**One care point, and it differs from ADR-072's change:** the vetting path is *not* a no-op here.
ADR-072's removal had zero behavioural effect on vetting because that call site already passed
`uuid.Nil` for `file_id`. Vetting chunks have a **real** `segment_id`, so this change alters their
signing input and needs its own test.

## Consequences

ADR-072's decision stands; its justification is replaced by one that holds independently of any
property of `chunk_id`. F-LTS-01 closes. F-LTS-02 closes on the DM constraint. R-60 closes; Domain H
reduces to R-59 (squatting) as a research note, plus R-72 (capability expiry and revocation) in the
new Domain W.

**Open constraints:**

- **Revocation remains unaddressed.** The token carries 8 bytes of expiry and no revocation path.
  A stolen token is valid until expiry against a provider that never contacts the microservice.
  → Domain W, R-72.
- **The generalisation is not applied elsewhere.** ADR-073's `chunk_id`, ADR-072's token, and
  Q68-3's authenticator all conflate content with position. Two are now fixed; the third is open.
  Domain W exists so the class is addressed rather than each instance.

## References

- ADR-072 — the decision this amends; unchanged
- ADR-073 — established the `chunk_id` definition that falsified ADR-072's argument
- ADR-019 — the source of the fresh-per-segment-key property §4 elevates to an invariant
- ADR-077 — trigger T3, which this failure motivated
- `reading-list.md` §6 Domain W — R-59, R-72, and the object-capability literature
