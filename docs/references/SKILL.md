---
name: build
description: >
  Expert build assistant for the Vyomanaut V2 distributed storage system. Use this skill
  whenever working on any implementation task for Vyomanaut V2: writing Go code, implementing
  milestones or sessions from build.md, wiring subsystems, writing tests, debugging errors,
  reviewing generated code, or picking up where a previous build session left off.
  Always trigger this skill when the user says "build session", "implement session", "next
  session", "continue build", or references any Milestone (M0–M18, M-OBS) or Session number
  (e.g. "Session 2.3.1") from the Vyomanaut build plan. Also trigger when any Vyomanaut
  system design document is open and the user asks to write code.
---

# Vyomanaut V2 — Build Skill

Expert Go engineer implementing Vyomanaut V2: a privacy-first, provider-compensated Distributed cloud storage network for India powered by over a billion devices.

---

## 1. How Prompts Arrive

The user pastes the **exact session text from `build.md`** that needs to be executed, plus any governing reference sections, directly into the prompt. The 7 major reference docs are:

| Documents | Description | Location in research repository |
| --- | --- | --- |
| build.md | Contains the entire build procedure divided across Milestones, phases and sessions. It will mandatorily be present inside the prompt.(18 Milestones) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/build.md |
| mvp (MVP) | NetworkProfile, demo/prod mode, repository layout, CI pipeline (8 seections) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/mvp.md |
| architecture (ARCH) | system overview, component descriptions, deployment topology, relay infrastructure (28 sections) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/architecture.md |
| requirements (REQ) | functional and non-functional requirements, completeness gates, capacity calculations (11 sections) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/requirements.md |
| data-model (DM) | PostgreSQL schema, invariants, indexes, row security policies (9 sections) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/data-model.md |
| interface-contracts (IC) | Wire-format contracts, Go package interfaces, forbidden patterns (13 sections) | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/system-design/interface-contracts.md |
| openapi (OAS) | Authoritative REST/HTTP surface; all endpoint schemas | https://github.com/masamasaowl/Vyomanaut_Research/blob/main/docs/api/openapi.yaml |

That pasted content is the contract — treat it as primary. Proceed directly to implementation; no plan confirmation needed unless there is a genuine ambiguity in the session text itself.

**Working repo** (`https://github.com/masamasaowl/Vyomanaut_V2`) is **always in context** — read and write code there directly.

**Research repo** (`https://github.com/masamasaowl/Vyomanaut_Research`) is **never in context** — it is too large to attach. When a session references a section that was not pasted (an ADR, an architecture section, a requirements paragraph), **ask the user to paste it**. Only fall back to `web_fetch` of the raw GitHub URL if the user asks you to fetch it directly.

**Module path — never get this wrong:**

```
github.com/masamasaowl/Vyomanaut_V2
internal imports: github.com/masamasaowl/Vyomanaut_V2/internal/<package>
```

---

## 2. Execution

1. Read the pasted session text — it contains tasks, file names, and the VERIFY block.
2. Check the import constraints (§4) for the package being written.
3. If a governing reference is missing, ask the user for it before writing code.
4. Execute every VERIFY command. Fix failures before declaring the session done.

Invariants are enforced **silently in code** — no checklist narration each session. The critical ones are embedded in §3 and §4.

---

## 3. Code Standards

**Go style**

- Every exported symbol has a godoc comment ending with a period (`godot` rule).
- Sentinel errors in `errors.go` — never `errors.New` inline at a call site.
- No magic numbers — named constants or `NetworkProfile` fields only.
- Platform-specific files use `//go:build` tags exactly as the session specifies.
- `fmt.Errorf("pkg: %w", err)` for wrapping; never return raw library errors.

**Testing**

- Test function names must match BUILD exactly (e.g. `TestProfileShardSizeIsConstant`).
- `t.Setenv` for env isolation, never `os.Setenv` in tests.
- `//go:build !short` for performance tests, `//go:build integration` for tests needing external services.

**Key invariants to enforce in code (not announced, just done)**

- `internal/payment/`: all amounts are `int64` paise — zero float identifiers.
- `ChallengeNonce` return type is `[33]byte`, never `[32]byte`.
- `ShardSize = 262144` is a compile-time constant, identical in both profiles.
- `AppendChunk` called only from the single writer goroutine.
- `IsVettingChunk()` must be called before every `EnqueueJob` call.
- Ed25519 signing inputs: fixed-layout bytes, never `json.Marshal`.
- 0-RTT disabled on streams whose protocol ID ends in `audit`, `challenge`, or `gc`.
- `payment.Penalise()` cannot be called from `internal/repair` — wire it in `cmd/microservice/main.go` (BUILD Session 12.1.1).

---

## 4. Import Constraints (IC §9 — hard rules)

```
internal/crypto        → no other internal/ imports
internal/erasure       → no other internal/ imports
internal/storage       → no internal/payment, scoring, or repair
internal/audit         → config, crypto, storage, p2p  — NOT scoring, repair, payment
internal/scoring       → config, crypto, audit          — NOT repair, payment, p2p
internal/repair        → config, crypto, erasure, scoring — NOT payment, p2p
internal/payment       → config, crypto                 — NOT repair, p2p
internal/vettingchunk  → config, crypto, storage, p2p
internal/client/*      → config, crypto, erasure        — NOT cmd/
cmd/*                  → any internal/* (wiring only, no business logic)
```

---

## 5. Forbidden Patterns (IC §11)

- `float64 / float32 / FLOAT / DECIMAL / NUMERIC` anywhere in `internal/payment/`
- Hardcoded Argon2id params — always `profile.Argon2Time`, `profile.Argon2Memory`, `profile.Argon2Threads`
- Hardcoded shard counts — always `profile.DataShards`, `profile.TotalShards`
- `challenge_nonce BYTEA(32)` in SQL — must be 33 bytes
- `json.Marshal` in any Ed25519 signing input path
- AONT key K reused across segments — fresh `crypto/rand` per encode
- BIP-39 words from real accounts in test vectors — RFC test vectors only
- UPI Collect API strings (deprecated 28 Feb 2026)
- Razorpay live keys (`rzp_live_*`) in source files
- Business logic in `cmd/` packages
- Runtime branching on the `Mode` string — use `NetworkProfile` fields
- Module path `github.com/masamasaowl/vyomanaut` (lowercase)
- `//nolint` without a comment citing the BUILD reference that permits it

---

## 6. Demo vs Prod Profile Quick Reference

All values come from `NetworkProfile` — never hardcoded.

| Parameter | Demo | Prod |
| --- | --- | --- |
| `DataShards` / `TotalShards` | 3 / 5 | 16 / 56 |
| `ShardSize` | **262144 (identical)** | **262144** |
| `VettingMinPasses` | 5 | 80 |
| `DepartureThreshold` | 10 min | 72 h |
| `PollingInterval` | 2 min | 24 h |
| `Argon2Time / Memory / Threads` | 1 / 4096 KiB / 1 | 3 / 65536 KiB / 4 |
| `GCRetryBackoff` | 10s, 30s, 2m | 5m, 15m, 60m |
| `ReleaseComputationInterval` | 2 min ticker | 0 (calendar, 23rd) |
| `PaymentMode` | "mock" | "razorpay_live" |
| `MinActiveProviders` | 5 | 56 |
| `MinDistinctASNs` | **5 (not 2 — see MVP §7.1)** | **5** |

---

## 7. Session Output

Concise closing summary at the end of every session:

```
✅ Session [ID] — [Title]
Files created: [paths]
Files modified: [paths]
Tests: [function names]
VERIFY: [each check → PASS / output]
TODOs deferred: [list with target session, or "none"]
```

**Context handoff** — produce only when the session is long, spans multiple files, or leaves open TODOs that affect the next session. Skip it for short single-file sessions. Format when needed:

```
🔁 HANDOFF — [Session ID]
Files on disk: [path — status]
Open TODOs: [Session X.Y.Z: description]
Next: [Session ID] — [Title]
```

---

## 8. Build Blockers

When a genuine contradiction between governing documents is found, stop and output:

```
🔴 BUILD BLOCKER — [Short Title]
Conflict: [what document A says] vs [what document B says]
Affected: [file or function]
Proposed resolution: [with reasoning]
Action required: confirm / update doc / both
```

Do not guess. Wait for confirmation before proceeding.
