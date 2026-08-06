# ADR-058 — Provider Daemon Local Keystore Key Derivation

**Status:** Proposed
**Topic:** #10 Key Management Strategy (provider-side gap) / #21 P2P Transfer Protocol (identity persistence)
**Supersedes:** —
**Superseded by:** —
**Research source:** requirements.md FR-023; ADR-020 (owner key hierarchy); ADR-021 §Peer identity
(keypair generated before registration); data-model.md §4.2 (`provider_id UUID PRIMARY KEY
DEFAULT gen_random_uuid()`); `internal/crypto/chacha20poly1305.go` (`EncryptAEAD`/`DecryptAEAD`);
`internal/p2p/identity.go`; `Vyomanaut_V2` checkout inspected directly for this ADR.

---

## Context

ADR-020 fully specifies the **data owner's** key hierarchy:
`Argon2id(passphrase, owner_id) → master_secret → HKDF-derived sub-keys`, including
`DeriveKeystoreEncKey(masterSecret, ownerID)` for the owner's local keystore. No equivalent
document specifies how a **provider daemon** encrypts its own local keystore. FR-023 says only
that the daemon must, at first launch, "generate an Ed25519 key pair, persist it encrypted under
a daemon-local passphrase" — it does not say what KDF is used, what the salt is, or where the
passphrase comes from.

**Half of the resulting defect has already been fixed independently of this ADR.**
`internal/crypto` now exposes `EncryptAEAD`/`DecryptAEAD` as a general-purpose AEAD primitive,
with `EncryptPointerFile`/`DecryptPointerFile` reimplemented as thin wrappers around it
specifically for the pointer-file artifact (code comment: *"the general-purpose AEAD primitive —
not EncryptPointerFile, which is reserved for the pointer-file artifact specifically... (M2 review
§3)"*). `internal/p2p/identity.go` correctly calls `EncryptAEAD`/`DecryptAEAD` to encrypt the
daemon's identity key on disk, with a random nonce — no `EncryptPointerFile`/`DecryptPointerFile`
misuse remains.

**The other half is still open.** `identity.go`'s `LoadOrGenerateIdentity` derives its local
keystore encryption key with:

```go
masterSecret := localcrypto.DeriveMasterSecret(passphrase, ownerID[:], ...)
encKey := localcrypto.DeriveKeystoreEncKey(masterSecret[:], ownerID)
```

This is the data-owner-scoped function, called with a provider daemon's key material — a provider
has no `owner_id`, and per ADR-021 the daemon generates this very identity **before** registering
with the microservice, i.e. before any `provider_id` exists either (`provider_id` is a
database-generated UUID assigned only at registration, `DEFAULT gen_random_uuid()`,
data-model.md §4.2). Whatever `ownerID`/`passphrase` values `identity.go` is actually supplying
here to satisfy the compiler, they cannot be a real data-owner identity — this is a placeholder
that has not been designed, not a real key hierarchy.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Continue reusing `DeriveKeystoreEncKey`/`DeriveMasterSecret` with placeholder owner-shaped inputs | No new code | Semantically wrong; doesn't solve the pre-registration bootstrapping problem; conflates two independent key hierarchies; whatever placeholder is currently satisfying the `ownerID`/`passphrase` parameters is undocumented and unauditable |
| Require the provider operator go through the full Argon2id/master-secret/BIP-39 flow, like a data owner | Reuses all existing machinery exactly | Providers have no concept of "recovering files" — imposing a 24-word mnemonic backup ritual on every provider install is disproportionate UX friction for a daemon whose only secret is one Ed25519 key |
| **A dedicated, provider-scoped local keystore key, salted by a locally-generated value that exists before registration (chosen)** | Solves the bootstrapping order problem; keeps the two hierarchies (owner vs. provider) explicitly separate in both naming and code; minimal new surface area; matches the precedent already set by `EncryptAEAD` being split out as its own general-purpose primitive rather than overloading `EncryptPointerFile` | One new function + one new on-disk salt file to manage |

## Decision

1. At first launch, before any Ed25519 key exists, the daemon generates a **16-byte random local
   salt** via `crypto/rand` and persists it in plaintext (a KDF salt is not required to be secret)
   alongside the encrypted keystore, e.g. `<data-dir>/keystore.salt`.
2. The daemon's local keystore encryption key is derived as:

   ```
   provider_keystore_enc_key = Argon2id(
       passphrase = daemon_local_passphrase,   // FR-023's "daemon-local passphrase"
       salt       = provider_local_salt,        // 16 random bytes, generated once, not an ownerID
       t, m, p    = profile.Argon2Time/Memory/Threads  // never hardcoded (ADR-031 discipline)
   )
   ```

3. Add `internal/crypto.DeriveProviderKeystoreEncKey(daemonLocalPassphrase, providerLocalSalt []byte, argon2Time uint32, argon2Memory uint32, argon2Threads uint8) [32]byte` — an explicitly
   provider-scoped sibling to `DeriveMasterSecret`, named so it can never be confused with the
   owner hierarchy at a call site.
4. `internal/p2p/identity.go`'s `LoadOrGenerateIdentity` is corrected to call this function instead
   of `DeriveMasterSecret`/`DeriveKeystoreEncKey`, dropping the `ownerID`/owner-passphrase
   parameters entirely — a provider daemon has neither. The existing `EncryptAEAD`/`DecryptAEAD`
   calls (already correct, see Context) are unaffected by this change.

## Consequences

**Positive:** closes the real bootstrapping-order bug; keeps the owner and provider key
hierarchies unambiguously separate in naming as well as design; brings the provider daemon's
local-secret handling under the same "never hardcode Argon2 cost params" discipline already
enforced elsewhere; completes the fix that `EncryptAEAD`/`DecryptAEAD` started.

**Negative / trade-offs:** one new function, one new small on-disk file (`keystore.salt`) to
include in backup/restore tooling and the daemon's file inventory (`mvp.md §8.2`,
`internal/p2p/**` entry — already updated to list `identity.go`'s role; add `keystore.salt` when
this ADR is implemented).

**Open constraint, not resolved by this ADR:** where the "daemon-local passphrase" itself comes
from — operator-entered at install time, or auto-generated and stored some other way — is a UX
decision. FR-023 should be amended to state which once decided.

**Scope note:** this ADR only proposes the M2-side crypto primitive
(`DeriveProviderKeystoreEncKey`). The actual `internal/p2p/identity.go` code change (Milestone 6)
is out of scope for the current documentation-alignment pass, which was bounded to Milestones 0–2;
it is recorded here so the M6 fix has a design to implement against.

## References

- FR-023 (requirements.md §4.5)
- ADR-020 (key hierarchy this ADR deliberately does *not* reuse)
- ADR-021 §Peer identity (keypair generated before registration — the reason the bootstrapping problem exists)
- data-model.md §4.2 (`provider_id UUID PRIMARY KEY DEFAULT gen_random_uuid()` — confirms no provider_id pre-exists)
- `internal/crypto/chacha20poly1305.go` — precedent for splitting a general-purpose primitive out from an artifact-specific wrapper
