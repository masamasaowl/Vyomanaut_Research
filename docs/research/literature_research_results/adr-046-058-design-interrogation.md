# Design Interrogation — ADR-046 to ADR-058

**Continues** [`adr-031-045-design-interrogation.md`](./adr-031-045-design-interrogation.md). Finding numbers continue from F-68. Final block.

| Route | Count |
| --- | --- |
| `[COUNCIL]` | 6 |
| `[RESEARCH]` | 2 |
| `[CORRECT]` | 5 |

**Headline:** the best-argued ADRs in the entire corpus are in this block. ADR-053 runs a 5,000-provider Monte Carlo, tests and rejects its own first design with a specific quantitative reason, and states the confidence it gave up. ADR-054 runs its cost model and reports that its own premise was wrong. ADR-047 resolves a genuine three-way conflict and corrects a precedent the project had been citing incorrectly. ADR-048 promotes a magic number to a profile field unprompted.

And one sentence in ADR-055 — written as reassurance — accidentally surfaces the largest confidentiality finding in the project: **RS repair assembles exactly the AONT plaintext threshold, so every repair event hands a complete plaintext-equivalent package to whoever performs it.** That is not a bug ADR-055 introduced. It has been true since ADR-004, on every repair, by construction.

---

## Band A — Blockers

### F-69 `[COUNCIL]` Every repair event reconstructs plaintext. The zero-knowledge property does not survive the repair protocol.

ADR-055 defends direct-to-owner delivery with:

> *"the AONT layer's key material is embedded in the package itself and does not require the owner's master secret to decode at the AONT layer, but the file itself remains protected by the owner's outer file-key encryption (ADR-020) — the microservice can reconstruct exactly what it could already reconstruct for ordinary repair."*

The second clause is correct and the first is the problem. **There is no outer file-key encryption.** ADR-022's pipeline is `plaintext → AONT (key K embedded) → RS`. Step 3 encrypts with `K = SecureRandom(256)`, not with `file_key`. ADR-020 describes `file_key` as *"for future re-encryption if the file is updated"* and stores it *inside* the pointer file — it is never applied to segment data. So an assembled AONT package decodes to plaintext with no owner secret at all. That is the entire point of AONT-RS, and it is why F-34 exists.

Follow the consequence through the repair protocol:

| Step | Requirement |
| --- | --- |
| AONT-RS plaintext threshold | any **16** of 56 shards |
| ADR-004 repair, per lost shard | *"contacts the k=16 surviving fragment holders, reconstructs"* |
| ADR-026, restating it | *"RS must download k entire chunks to reconstruct one"* |

**Repairing a single lost shard requires assembling exactly the plaintext threshold.** Whoever performs repair — microservice or a delegated provider, ADR-004 is ambiguous on which — holds a complete, decodable AONT package for that segment. Not under collusion, not under attack: by design, on every repair, at the lazy-repair cadence.

ADR-055's own defence is therefore right in the way it did not intend: *"the microservice can reconstruct exactly what it could already reconstruct for ordinary repair"* is true, and what it can already reconstruct is the plaintext.

This reframes three earlier findings. F-34 (two colluding ASNs) is the *adversarial* path to 16 shards; this is the *routine* one. F-11 and F-48 both flagged the zero-knowledge claim as overstated on metadata grounds; this shows it fails on content too. ADR-019, ADR-020 and ADR-022 all assert zero-knowledge as a property of the system, and Paper 18 (Tahoe-LAFS) is cited as confirming *"the zero-knowledge property holds across single-operator and federated topologies"* — but Tahoe does not have a coordinator-driven repair path that reassembles the ciphertext threshold.

**This is a solvable problem with real literature behind it** — secure/repair-preserving regenerating codes, and repair-by-transfer constructions where a helper contributes to reconstruction without any party assembling `k` shards. None of it is in the corpus. ADR-026 defers regenerating codes to V3 on *bandwidth* grounds, never noticing they also carry the confidentiality property the system claims to already have. → new reading-list Domain P.

At minimum the ADRs must stop claiming a property the repair protocol does not deliver. At best, this changes the V3 code-family decision (F-44) from a bandwidth question into a confidentiality one.

---

### F-70 `[COUNCIL]` ADR-053's confidence threshold tolerates exactly zero audit failures, and the ADR does not say so.

ADR-053 is the most rigorous ADR in the corpus. It rejected its own full-history design because a single fail requires **148** subsequent clean audits to recover to 0.99 confidence — which reproduces exactly: `(0.5+p)/(2+p) ≥ 0.99 → p ≥ 148`. Correct, well caught, well argued.

The same arithmetic applied to the *adopted* design has a consequence the ADR does not state. The rolling window holds at most **30** audits (30 days at 24-hour polling). Solving `(0.5+p)/(1+p+f) ≥ 0.97`:

| Fails in window | Passes needed | Total audits | Fits in a 30-slot window? |
| --- | --- | --- | --- |
| 0 | 16 | 16 | yes |
| **1** | **48** | **49** | **no — impossible** |
| 2 | 81 | 83 | no |

**At target 0.97 with a 30-audit window, a single failure anywhere in the trailing 30 days makes graduation mathematically impossible until it ages out.** The rule is not a confidence gradient; it is *"30 consecutive clean days"* wearing Bayesian clothing. The Monte Carlo results are consistent with this — 15.3% of the 97%-reliability tier forced to the ceiling is close to the chance of never seeing a clean 30-window — so the simulation is right; only the *description* is misleading.

Two live consequences:

1. **The threshold is a cliff, not a dial.** At 0.97 zero fails are tolerated; at 0.95 exactly one is (`28p + 1f = 29 ≤ 30`); at 0.93, one; at 0.90, two. Anyone tuning this post-launch — and ADR-053 explicitly expects tuning — will find behaviour jumping discontinuously between adjacent-looking values.
2. **A provider who fails once after day 60 is guaranteed forced graduation**, since the fail cannot age out before the 90-day ceiling. Under ADR-054 that provider's pay is capped at `C(t) ≤ 0.9516` for the remainder, and under F-71 it then *drops* at graduation.

None of this makes the design wrong. It makes the design a streak counter with a 30-day forgiveness window, which is a perfectly good mechanism — and should be documented as one, with the fail-tolerance table above stated explicitly so the next person to touch the threshold knows what they are touching.

---

### F-71 `[COUNCIL]` The earnings ramp and the post-graduation ladder are discontinuous, and forced graduation is a pay cut.

ADR-054 sets `RampMultiplier(t) = C(t)` — continuous, rising, day-zero value 0.5 matching ADR-024 exactly. Elegant, and the reuse of one signal for two purposes is the right instinct.

At graduation, control passes to ADR-024 §3's step function over `score_30d`: `≥0.95 → 1.00`, `≥0.80 → 0.75`, `≥0.65 → 0.50`, `<0.65 → 0.00`.

For a clean graduate this is fine (C ≈ 0.984 → score ≈ 0.98 → 1.00×; a small raise). For a **forced** graduate it inverts:

| Provider | Day 89 (ramp) | Day 90 (ladder) | Effect |
| --- | --- | --- | --- |
| Borderline, C ≈ 0.904 | 0.904× | 0.75× | **−17%** |
| Poor, C ≈ 0.744 | 0.744× | 0.50× | **−33%** |

A provider is told they have graduated — ADR-054 specifies *"a plain notice that full rate has started"* — and their next payout falls by a third. Under ADR-045's disclosure policy they get a reason category and no formula, so the message is "you graduated to full rate" followed by less money.

Both ADRs are individually coherent; neither owns the seam. The fix is small — carry the ramp through a transition window, or align the ladder's top rung with the ramp's exit value — but it has to be decided by whoever owns both, and ADR-054's supersession note explicitly leaves ADR-024 §3 untouched.

---

### F-72 `[COUNCIL]` ADR-047 stops the daemon at logoff, and the availability model assumes it doesn't.

ADR-047 resolves the FR-031 / ADR-040 / ADR-042 three-way conflict correctly, and the Session 0 reasoning is right: a service cannot render a tray icon, and installing one needs admin. Correcting the Tailscale precedent — Tailscale *does* run an admin-installed Session 0 service, and pays that cost because it drives a virtual network adapter, which Vyomanaut does not — is a genuinely good catch.

But the accepted consequence is:

> *"the daemon does not run before any user logs on, and stops when the user logs off — acceptable for this product, since a machine with nobody logged in has no interactive desktop for the tray to serve anyway."*

That reasoning conflates the **tray's** need for a session with the **daemon's** need to serve chunks. Responding to audits and serving retrievals has nothing to do with whether a human is looking at an icon.

The practical case is Windows Update. Automatic reboots are frequent, land overnight, and leave the machine at the login screen. A machine that reboots at 03:00 and is logged into at 08:00 is, for audit purposes, **offline for five hours** — powered on, network up, disk intact, daemon not running. At 24-hour polling that is a real chance of a TIMEOUT, which decrements the 24-hour score (ADR-008), which under ADR-024 §3 can move the release multiplier, which under ADR-045 the provider sees as a smaller payout with a reason category.

That is precisely the failure mode ADR-028 was written to eliminate — *"the provider begins losing earnings despite being consistently available… they perceive this as a payment bug and churn"* — arriving through a different door, and ADR-028's 4-hour heartbeat does not help because the daemon isn't running to send one.

Note the interaction with ADR-057: sleep inhibition covers lid-close while logged in, and does nothing for logoff or post-reboot. So the two ADRs together cover the laptop-lid case and leave the more common desktop case open.

Options exist that keep ADR-042 intact — `schtasks` also supports a boot trigger under the user account with "run only when user is logged on" unset, though that requires stored credentials and reintroduces elevation; or accept the gap and exclude logged-out time from scoring, which is a scoring change rather than a mechanism change. Either way, the MTTF assumption of 180–380 days is currently being applied to a daemon whose real duty cycle is *time-a-human-is-logged-in*, and nobody has measured that.

---

## Band B — Corrections and consequences

### F-73 `[CORRECT]` BadgerDB defaults to `SyncWrites: false`, and ADR-046's config block does not override it.

ADR-046's risk analysis is sound — the MSVC/MinGW gap is real, facebook/rocksdb#3984 shows even a successful compile isn't a correct binary, and a pure-Go engine implementing the same WiscKey design is the right escape.

The configuration block sets `ValueThreshold`, `BloomFalsePositive`, `BlockCacheSize` and `Compression`, and the ADR states these are chosen *"to reproduce the properties ADR-023's audit-latency budget depends on, not Badger's general-purpose defaults."* It omits the durability one.

ADR-023's write path is explicit: *"`fsync()` the vLog before proceeding"* then insert into RocksDB, so that *"if the daemon crashes between steps 3 and 4, the vLog entry is durable."* Badger v4's `DefaultOptions` set `SyncWrites: false`. Without `WithSyncWrites(true)`, a Windows provider that acknowledges a chunk store and then loses power can come back without the chunk — having already signed the transitory storage receipt (ADR-002) confirming it was stored.

That is a cross-platform durability divergence in the one property the audit system is built to detect, and it is silent: the provider reports the chunk stored, then fails the audit days later, and the reliability scorer treats it as provider misbehaviour.

Related, smaller: ADR-023's GC fires only on deletion events; Badger's `RunValueLogGC` is caller-scheduled with a discard-ratio argument. The `RunGC()` interface hides this, but *when* space is reclaimed differs, which matters for the HDD compaction benchmark ADR-046 correctly says must be re-run.

---

### F-74 `[CORRECT]` ADR-051 Phase 1 verifies integrity, not authenticity, and never says what actually protects it.

Phase 1 downloads an NSIS installer from GitHub Releases, verifies it *"against a published SHA-256 checksum,"* and executes it silently via `/S`.

A SHA-256 published alongside the artifact it hashes is an integrity check against corruption, not authenticity: anyone who can replace the release asset can replace the checksum. The ADR notes Phase 2 uses *"SHA-256 plus Ed25519 signature verification"* — so it knows the difference and ships the weaker form first, in an auto-update path that executes an installer.

In practice this is probably closed, because ADR-041 requires every installer to be Authenticode-signed via Azure Trusted Signing. But ADR-051 never mentions Authenticode, so the guarantee is accidental rather than designed — and `/S` silent execution may suppress the very prompt that would surface a signature failure. Phase 1 should verify the Authenticode signature programmatically before executing, and the ADR should say that is where the authenticity comes from.

**Also unresolved and connected:** Q53-1 asks whether the `/S` reinstall triggers UAC. ADR-051 frames this as a question about whether "silent" holds. It is also an ADR-042 question — *"no elevated or administrator privileges for any of its functionality, at any point."* If the reinstall needs elevation, Phase 1 does not merely feel less silent; it violates a standing constraint. The two ADRs should be reconciled before Phase 1 ships.

---

### F-75 `[COUNCIL]` ADR-058 defers the question that determines whether the provider keystore protects anything.

ADR-058 correctly identifies a real bug: `identity.go` derives the daemon's keystore key with `DeriveMasterSecret`/`DeriveKeystoreEncKey` — owner-scoped functions called with placeholder inputs, for an identity generated *before* any `provider_id` exists. The fix (a provider-scoped derivation salted by a locally-generated value) is right, and splitting the function by name so it cannot be confused at a call site follows the `EncryptAEAD` precedent well.

The open constraint is where the security actually lives:

> *"where the 'daemon-local passphrase' itself comes from — operator-entered at install time, or auto-generated and stored some other way — is a UX decision."*

It is not a UX decision. The two options have opposite security properties, and both conflict with something:

- **Auto-generated and stored locally** — the encryption key material sits on the same disk as the ciphertext it protects, so the keystore is encrypted against nothing. The daemon restarts unattended, satisfying ADR-047 and ADR-009.
- **Operator-entered** — real protection, but the daemon cannot start without a human at the keyboard. That breaks ADR-047's logon-trigger autostart, ADR-009's background execution, and ADR-051's silent-update relaunch simultaneously.

An always-on unattended daemon cannot have an interactive secret, and a non-interactive local secret is not a secret against local disk access. The standard resolution is OS keystore integration — Windows DPAPI (`CryptProtectData`, user-scoped, no elevation, which suits ADR-042), macOS Keychain, Linux Secret Service — where the OS binds the key to the user account rather than the filesystem. None of the three is mentioned in ADR-058 or anywhere else in the corpus.

Secondary: the ADR derives with `profile.Argon2Time/Memory/Threads`, which are the *owner's* interactive-login parameters (t=3, m=64 MB). If the daemon passphrase ends up machine-generated and high-entropy, Argon2 is unnecessary and HKDF suffices; if human-chosen, those parameters are roughly right. The parameter choice is downstream of the unresolved question, so it should not be locked in first.

---

### F-76 `[CORRECT]` ADR-048 secures gossip against forgery and leaves F-35's divergence untouched — while naming the mechanism.

ADR-048 is thorough. Per-replica Ed25519 rather than reusing the shared audit HMAC is the right call for the right reason (*"it authenticates 'a member of the cluster,' never which member"*), the secrets-manager path convention reuses existing machinery, and the rejection of mTLS on consistency grounds is well argued.

It also states what F-35 was asking about:

> *"`HealthyCount()` and `ResponsibleReplica()` drive which replica is treated as authoritative for audit-challenge dispatch and chunk-assignment decisions."*

So `ResponsibleReplica()` **is** the mechanism selecting the single authoritative writer — and it is a pure function of gossip membership, which is eventually consistent with a 1-second cadence and a 5-second healthy window, refreshed at clients every 10 seconds (ADR-025).

Authentication prevents a *forged* membership claim. It does not prevent two replicas holding *divergent but individually genuine* views and each computing a different responsible replica for the same key. For escrow debit — a bounded counter with a `≥ 0` floor, explicitly non-I-confluent in ADR-013 — that is a double-release path, and no fencing token exists to make the loser's writes rejected.

F-35 stands unchanged and is now more concrete: the design has consistent-hashing-style routing where it needs leader election with fencing. Reading-list Domain L (R-30, R-31) is the right response and is unaffected.

Worth noting positively: ADR-048 promotes `GossipHealthyWindow` from a hardcoded 5 seconds to a profile field *and states its derivation* (5× the 1-second gossip cadence, tolerating four missed cycles). That is exactly the F-54 discipline, applied without being asked.

---

### F-77 `[CORRECT]` F-42's vetting-income stack improves — and what remains is entirely ADR-030's storage cap.

ADR-053 and ADR-054 together substantially fix F-42. Recomputed:

| Design | Duration | Release multiplier | × ADR-030 10% cap | Share of steady-state income |
| --- | --- | --- | --- | --- |
| ADR-005 + ADR-024 §6 (old) | 150 d | flat 0.50 | 0.10 | **5%** for ~5 months |
| ADR-053 + ADR-054, flawless | 45 d | ramp ≈ 0.75 avg | 0.10 | **8%** for ~1.5 months |
| ADR-053 + ADR-054, borderline | 90 d | ramp ≈ 0.70 avg | 0.10 | 7% for 3 months |

Duration roughly thirds and the multiplier rises, so the provider-facing experience improves markedly. ADR-054's honesty about this — running the cost model and reporting that its own "accept extra cash burn" framing *"does not hold up numerically… the honest finding is closer to the opposite"* — is the kind of thing that should happen more often and almost never does.

**The residual is now dominated by ADR-030's 10% synthetic-chunk storage cap**, which neither ADR revisits. A provider offering 200 GB still earns on 20 GB throughout vetting. That cap exists for a good reason (don't commit an unproven provider's disk to dummy data) but it is now the binding term in the onboarding-income equation, and it should be reconsidered alongside the ramp rather than left as the one untouched multiplier.

**And ADR-054 removed a deterrent without replacing it.** Vetting-stage earnings are no longer seized on departure, on the reasoning that vetting providers hold only synthetic chunks so no real-data exposure exists. Correct as far as it goes — but combined with a *rising* ramp, the Sybil-farm play is now: register, behave perfectly for 45 days, collect a ramp reaching ~0.98×, depart with no forfeiture, repeat. Both ADR-053 and ADR-054 flag the exposure and both explicitly decline to compute the bound. That bound is now materially more urgent than when it was first logged.

---

### F-78 `[CORRECT]` ADR-052 confirms F-56's concern rather than resolving it.

ADR-052 is a good decision — Paraglide's compile-time approach matches ADR-049's reasoning, and `Intl.NumberFormat("en-IN", { style: "currency", currency: "INR" })` is genuinely the correct fix for lakh/crore digit grouping, shipped unconditionally and independent of the deferred language-scope question. Both halves are right.

But it specifies that copy is *"authored as keyed Paraglide JS messages, sourced from `interface-contracts.md` §14's copy table once it merges."* So the pipeline is now:

```
IC §14 markdown table  →  (hand-transcribed)  →  Paraglide message files  →  compiled functions
```

F-56 already noted ADR-034 gives the table three possible homes. ADR-052 adds a fourth artifact downstream, with no mechanism keeping it in sync — and ADR-034's process rule (*"adding a new ErrorCode should add a row to IC §14 in the same change"*) now needs to become a three-way reconciliation across Go, IC and Paraglide.

ADR-034 rejected Alternative C (a machine-readable source compiled into the UI) partly because a markdown table is reviewable without a Go toolchain. That reasoning was sound in isolation and is weaker now: Paraglide message files are JSON, equally reviewable, and would be a source rather than a transcription target. ADR-034 anticipated this — *"can be mechanically compiled into a Go map or JSON asset at build time… worth revisiting once M15 is underway"* — and ADR-052 is the trigger it was waiting for.

---

### F-79 `[RESEARCH]` ADR-050's baseline is correctly chosen and has no revision trigger.

Selecting WCAG 2.1 AA rather than 2.2 *because* IS 17802 and GIGW 3.0 reference 2.1 is the right reasoning — conformance follows the referenced standard, not the newest one. Grounding the decision in RPwD Act 2016 obligations, and specifying a concrete check (Windows Narrator against the packaged Wails build, not markup inference) against a documented WebView2 accessibility-tree gap, makes this one of the better-instrumented UX decisions in the corpus.

Two small gaps. There is no recorded trigger for revisiting if GIGW or IS 17802 update their referenced version — and unlike the Wails v3 triggers (ADR-038, ADR-040, ADR-051), this one depends on an external regulator's timetable that nobody is watching. And the Narrator smoke test covers Windows/WebView2 only; the ADR notes an open Wails accessibility gap on Linux/WebKitGTK, which is correct sequencing but means the constraint has to be re-checked, not inherited, when those platforms are scoped.

---

## What holds up

This block is the strongest in the corpus and it deserves to be said plainly.

- **ADR-053** ran a 5,000-provider Monte Carlo across four reliability tiers, **tested and rejected its own first design** with a reproducible number (a single fail requires 148 clean audits to recover at 0.99 full-history — verified exactly), recalibrated the target with a stated reason, quantified the confidence it gave up (98.4% vs 99.4%), and named the hysteresis edge case it does not solve. Nothing else in the project is argued to this standard.
- **ADR-054** built its cost model and then reported that its own motivating premise was wrong, in its own text, rather than quietly dropping the comparison. It also flagged the assumption that comparison rests on and asked for it to be re-run if that assumption is false.
- **ADR-047** resolved a real three-way conflict between an FR and two ADRs, and corrected a precedent (Tailscale) the project had been citing for something it does not do. F-72 is about a consequence it accepted too quickly, not about the resolution, which is right.
- **ADR-048** extended ADR-036's pattern one layer up without re-deriving its constants, rejected two plausible alternatives with specific reasoning, and promoted a magic number to a profile field with its derivation recorded.
- **ADR-055** derived `θ = ⌈0.75 × ParityShards⌉` instead of hardcoding 30, then checked it two independent ways — ASN-cap consistency (30 failures requires ≥3 ASNs under the 20% cap, so it is definitionally multi-region) and statistical implausibility under independence. That second figure is if anything conservative: the true value is ~10⁻⁷⁶, not 10⁻⁵⁰.
- **ADR-058** found a live bug by reading the actual checkout, and correctly identified that half of it had already been fixed independently.
- **ADR-046** and **ADR-057** each corrected an error in their own originating proposal before implementation — the RocksDB/MinGW runtime failure, and PowerToys Awake not handling lid-close.

The pattern across this block — derive rather than assert, check the derivation twice, report when your own premise fails — is the thing worth institutionalising.

---

## Routing summary

**To the council:**

1. **F-69** — repair reconstructs plaintext. Ranks with F-32 and F-34; it is the third structural finding and the only one that is true on every ordinary operation rather than under attack.
2. **F-72** — the daemon stops at logoff. Decide whether to change the mechanism or exclude logged-out time from scoring.
3. **F-75** — where the daemon-local passphrase comes from. Not a UX decision; it determines whether the keystore protects anything.
4. **F-70 / F-71** — document ADR-053's zero-fail tolerance and its threshold cliff; close the ramp-to-ladder seam so forced graduation is not a pay cut.
5. **F-77** — ADR-030's 10% cap is now the binding term in vetting income, and the Sybil bound needs computing now that seizure is gone.

**Corrections:** F-73 (Badger `SyncWrites`), F-74 (Authenticode in ADR-051 Phase 1, and its ADR-042 collision), F-76 (F-35 restated against `ResponsibleReplica()`), F-78 (revisit ADR-034 Alternative C), F-79 (ADR-050 revision trigger).

**To the reading list:** F-69 opens **Domain P — confidentiality-preserving repair**, the last new domain. F-75 adds an OS-keystore topic to Domain M. F-70 adds nothing — it is arithmetic, already done here.
