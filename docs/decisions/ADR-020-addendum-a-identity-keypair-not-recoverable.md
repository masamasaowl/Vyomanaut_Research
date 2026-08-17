# ADR-020 Addendum A — Master-Secret Recovery Does Not Restore the Ed25519 Identity Key

**Status:** Accepted
**Track:** Both — ratifies live demo-track CLI behavior (M17 Session 17.1.1); the LTS gap (no re-keying endpoint) is recorded, not resolved, here
**Topic:** #10 Key Management Strategy / #14 Client Interface
**Amends:** ADR-020 (the three-level HKDF hierarchy and pointer-file recovery stand; the "Full recovery" claim in Backup and Recovery Paths Scenarios 1–2 does not, for the Ed25519 identity key specifically)
**Research source:** Design Council verdict "Owner Registration: Keypair/OwnerID Ordering" (M17 Session 17.1.1), Systems Theorist seat; live-tracing `internal/client/account`, `internal/api/owner.go`, `internal/api/otp.go` against ADR-020's own text

---

## The claim being withdrawn

ADR-020's "Backup and Recovery Paths" section states, for both **Scenario 1** (device loss, passphrase known) and **Scenario 2** (device loss, mnemonic available): *"Daemon re-derives master_secret and all file keys... Full recovery."* That claim holds for everything the three-level hierarchy ADR-020 actually specifies — master_secret, file_key, pointer_file_enc_key, and (via AONT-RS's own re-derivation from k=16 shards) the AONT key K. **It does not hold for the Ed25519 identity keypair.**

ADR-020's own text already names this key as structurally separate — *"The pointer file is signed by the data owner's Ed25519 key (separate from encryption keys, used for pointer file integrity)"* — but its Backup and Recovery Paths section never revisits that separateness once it reaches the actual recovery scenarios, and folds the Ed25519 key into "Full recovery" by omission rather than by an argument that it's actually covered.

As built (M15 `internal/client/account`; M17 Session 17.1.1's live wiring):

- The master secret is Argon2id-derived from `(passphrase, owner_id, profile.Argon2*)`, or reconstructible from the 24-word mnemonic — deterministic, exactly as ADR-020 specifies. Either input always reproduces it.
- The Ed25519 identity keypair is generated once, via `crypto/rand`, at registration (`account.RegisterOwner`) — **not derived from the master secret, the passphrase, or the mnemonic by any path.** It exists **only** as the encrypted keystore (`account.Keystore`, AEAD-protected under `DeriveKeystoreEncKey(masterSecret, ownerID)`) on the device that registered it. The server retains only the **public** key (`owners.ed25519_public_key`) — it never held the private half and has no way to return it.
- This key is not incidental to account registration alone: `internal/client/upload.NewOrchestrator`'s `signingKey` parameter — documented as `internal/client/account.Identity.PrivateKey` — signs `owner_sig` on **every** file registration, not just the original registration call. Losing this key doesn't just affect one signature; from that point on it removes the ability to register any new file under this `owner_id`, permanently.
- Checked against every owner-scoped route in `internal/api/router.go` (`register`, `deposit`, `balance`, `files`, `escrow`, `withdraw`): **there is no endpoint that lets an owner register a replacement public key against an existing `owner_id`.**

So: *"passphrase or mnemonic"* recovers the master secret and everything the master secret protects. It does not, and structurally cannot, recover the Ed25519 identity key unless the original encrypted keystore file is also physically available to decrypt with the recovered master secret. If both the device and any backup of that keystore file are gone, the private key is gone — permanently, by construction, regardless of how many valid recovery credentials the owner presents.

## Why this is correct, not a bug to fix by deriving the key

The obvious-looking fix — derive the Ed25519 key from the master secret too, the way `file_key` and `pointer_file_enc_key` already are (ADR-020 Level 2 / pointer-file derivation) — was considered and rejected. It would collapse two independent security domains into one secret: today, a compromised or guessed mnemonic gives an attacker read access to the owner's existing files (bad, but bounded — confidentiality only), without also handing them the ability to sign new uploads or otherwise act as the owner going forward (authentication). Deriving the identity key from the master secret would mean the same single guess grants both. That is a strictly worse security posture, not a recovery convenience — a deliberate, structural property of the current design, which this addendum makes explicit rather than treats as something to fix.

## Decision

1. **ADR-020's Backup and Recovery Paths, Scenarios 1 and 2, are corrected in place**: "Full recovery" is replaced with an explicit split — master-secret-derived credentials (file decryption, pointer files) are always recovered; the Ed25519 identity key is recovered **only if** the original encrypted keystore is also available to decrypt. This corrects what the existing mechanism actually guarantees; it introduces no new mechanism.
2. **The two-secret separation is a ratified invariant**, joining this project's existing "silent invariants" list (build.md §3: AONT key K never reused across segments, `ChallengeNonce` always `[33]byte`, etc.): the Ed25519 identity keypair must never be derived from the master secret, the passphrase, or the mnemonic, and vice versa. A future code path that does so is a correctness violation of this addendum, not a legitimate optimization.
3. **`cmd/client recover`'s live behavior (Session 17.1.1) is this addendum's demo-track ratification, not a stopgap.** Network recovery (phone+OTP login — `internal/api/otp.go`'s `HandleVerify` already returns a fresh JWT and `entity_id` for an existing phone number, with no further registration call needed) always restores account access and master-secret-dependent capability. It restores signing capability only when the original keystore is also decryptable, and reports which case occurred explicitly (`signing_key_restored` in `--json` output; an explicit note in human-readable output) — it never silently claims full recovery when it wasn't.
4. **FR-004 needs a wording correction**, listed here for your approval rather than applied: *"restore access on a new device using either their passphrase or their 24-word BIP-39 mnemonic"* should state that this restores file confidentiality and account access, but not upload/signing continuity, absent the original keystore.
5. **A real fix — letting an owner register a replacement Ed25519 public key against their existing `owner_id` — is named as an explicit LTS open item, not designed here.** No such endpoint exists today (confirmed above against every owner route). Q-ADR20-1 below is the open question this leaves.

## Consequences

**Positive:** the mnemonic-compromise-≠-account-takeover property is now an explicit, documented, checkable invariant, instead of an accident of how M15's `account` package and M17's live wiring happened to compose. `recover`'s honest partial-success reporting has a stated rationale instead of reading like an unfinished feature.

**Negative / trade-offs:** FR-004 as currently worded overclaims and needs a correction (§4, pending approval). A real product gap persists — an owner who genuinely loses their keystore with no backup permanently loses the ability to upload new files under that identity — until the LTS re-keying work in Open Constraints closes it. Onboarding UX needs to actually disclose that the keystore file itself must be backed up, not just the mnemonic — an extension of the master-secret-loss disclosure ADR-020's own Open Constraints already called for, now needed for keystore loss specifically.

**Open constraints:**

- **Re-keying protocol design.** Q-ADR20-1: what proof should be required beyond a phone+OTP login before accepting a new Ed25519 public key for an existing `owner_id`? Phone-based auth is already this system's entire Sybil defense (FR-001) — using it alone as the bar for re-keying, a high-value action (it grants future signing authority over the account), may not be sufficient. LTS, not started.
- **Keystore export/backup as a first-class CLI feature.** Q-ADR20-2: should `cmd/client` grow an explicit `export-keystore`-style subcommand so owners have a supported way to carry their identity key to a new device, rather than relying on ad hoc file copying of `identity.json`? Not scoped in MVP §8.3's eight-subcommand table.
- **FR-004's wording correction (§4) is pending your approval**, not applied to `requirements.md` by this addendum.

## References

- [ADR-020](./ADR-020-key-management.md) — the three-level key hierarchy this addendum corrects the recovery claims of; everything else in it (HKDF derivation, pointer-file recovery, nonce-counter handling) is unaffected
- Design Council verdict "Owner Registration: Keypair/OwnerID Ordering" (M17 Session 17.1.1) — the Systems Theorist seat that first raised this as a documentation gap rather than purely a code question
- `internal/client/account/register.go`, `keystore.go`, `registerflow.go` — the live key-generation and keystore-encryption code this addendum describes
- `internal/api/owner.go` (`HandleRegister`), `internal/api/otp.go` (`HandleVerify`'s login case) — confirms the server never holds the Ed25519 private key and that no re-keying endpoint exists
- FR-004 (`requirements.md`) — the requirement whose wording needs correction per Decision §4
- `build.md` §3 — the "Silent invariants" list this addendum's item 2 joins
