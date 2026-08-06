---
name: build
description: >
  Expert build assistant for the Vyomanaut V2 distributed storage system. Use this skill
  whenever working on any implementation task for Vyomanaut V2: writing Go code, implementing
  milestones or sessions from build.md, wiring subsystems, writing tests, debugging errors,
  or picking up where a previous build session left off. Trigger on "build session",
  "implement session", "next session", "continue build", or any Milestone (M0–M18, M-OBS)
  or Session number (e.g. "Session 2.3.1") from the Vyomanaut build plan.
---

# Vyomanaut V2 — Build Skill

Expert Go engineer implementing Vyomanaut V2: a privacy-first, provider-compensated distributed cloud storage network for India.

---

## 1. Context

**Working repo** (`Vyomanaut_V2`) is in project context — scan existing files before writing anything.
**Research repo** (`Vyomanaut_Research`) is NOT in context — ask the user to paste needed sections; only `web_fetch` the raw GitHub URL if they explicitly ask.

The user pastes the **exact session text from `build.md`** plus any required reference sections. That pasted content is the contract. The six reference doc tags are:

| Tag | Document |
|---|---|
| **IC** | Interface contracts — wire formats, Go interfaces, forbidden patterns |
| **DM** | Data model — PostgreSQL schema, invariants, row security policies |
| **ARCH** | Architecture — system design, component details, deployment topology |
| **REQ** | Requirements — FRs, NFRs, completeness gates |
| **MVP** | NetworkProfile, demo/prod mode, repo layout, CI pipeline |
| **OAS** | REST/HTTP endpoint schemas (authoritative) |

**Go version:** The project targets Go 1.26, but the build environment only has **Go 1.22**. Write strictly 1.22-compatible code. Never use post-1.22 language features or `//go:build go1.26` guards. All constructs in this project are 1.22-compatible.

---

## 2. Execution Protocol

**Step 1 — Scan existing files.**
Read the `Vyomanaut_V2` project context to understand what is already on disk. Never reproduce a file that already exists. Pre-existing package files from prior sessions are correct and already compiled — trust them.

**Step 2 — Identify outputs.**
The session's `FILE:` blocks define the complete list of files to create or modify. Nothing outside those blocks belongs in the output.

**Step 3 — Engineering Review (one deliberate thinking step).**
Before writing a single line of code, pause and evaluate:
- Does the session's specified approach fit the existing packages on disk without friction or hidden conflicts?
- Is the algorithm complete, or are there edge cases the session text didn't address (e.g. nil handling, concurrent access, partial failure)?
- Is there a simpler or more idiomatic Go implementation that still satisfies every `FILE:` constraint and `VERIFY` requirement?

If yes to any: note the improvement concisely in 1–2 sentences, then proceed with the better approach while remaining fully spec-compliant. If nothing needs changing, move on without narrating it.

**Step 4 — Check imports.**
Verify the session's target package against §4 before writing.

**Step 5 — Write files.**
Use `create_file` for net-new files. Use `str_replace` for targeted edits to existing files. Never `create_file` a path that already exists on disk.

**Step 6 — Compile.**
Run `go build` (on the path specified in the session). All prior-session dependencies are already on disk; only the current session's new files need to be written.

**Step 7 — Run VERIFY.**
Execute every shell command in the `VERIFY` block. Fix genuine failures. Declare done only when all checks pass.

---

### On grep counts

`grep` counts every match including doc-comment lines. An exact `EXPECT: N` includes comment occurrences. **Never shorten, reword, or remove a doc comment solely to make a count match.**

---

## 3. Code Standards

- Sentinel errors in `errors.go` — never `errors.New` inline at a call site.
- No magic numbers — named constants or `NetworkProfile` fields only.
- `//go:build` tags exactly as specified in the session. Always create the `_other.go` stub alongside any tagged file.
- `fmt.Errorf("pkg: %w", err)` for wrapping; never return raw library errors.
- Test function names must match the session text exactly. `t.Setenv` for env isolation, never `os.Setenv`.
- `//go:build !short` for performance tests; `//go:build integration` for tests requiring external services.

**Silent invariants — enforced in code, never narrated per session:**

- `internal/payment/` — all monetary amounts are `int64` paise; zero float identifiers.
- `ChallengeNonce` return type is `[33]byte`, never `[32]byte`.
- `ShardSize = 262144` — compile-time constant, identical in demo and prod profiles.
- `AppendChunk` — single writer goroutine only; never called from concurrent goroutines directly.
- `IsVettingChunk()` — must be called before every `EnqueueJob`.
- Ed25519 signing inputs — fixed-layout byte sequences, never `json.Marshal`.
- 0-RTT policy — explicit protocol deny-list (not suffix or pattern matching).
- `payment.Penalise()` — wired in `cmd/microservice/main.go` (Session 12.1.1), never called from `internal/repair`.

---

## 4. Import Constraints (IC §9 — hard rules)

```
internal/crypto        → no other internal/
internal/erasure       → no other internal/
internal/storage       → NOT payment, scoring, repair
internal/audit         → config, crypto, storage, p2p  — NOT scoring, repair, payment
internal/scoring       → config, crypto, audit          — NOT repair, payment, p2p
internal/repair        → config, crypto, erasure, scoring — NOT payment, p2p
internal/payment       → config, crypto                 — NOT repair, p2p
internal/vettingchunk  → config, crypto, storage, p2p
internal/client/*      → config, crypto, erasure        — NOT cmd/
cmd/*                  → any internal/* (wiring only; no business logic)
```

---

## 5. Forbidden Patterns (IC §11)

- `float64 / float32 / FLOAT / DECIMAL / NUMERIC` anywhere in `internal/payment/`
- Hardcoded Argon2id params — always `profile.Argon2{Time,Memory,Threads}`
- Hardcoded shard counts — always `profile.DataShards`, `profile.TotalShards`
- `challenge_nonce BYTEA(32)` in any SQL — must always be 33 bytes
- `json.Marshal` in any Ed25519 signing input path
- AONT key K reused across segments — fresh `crypto/rand` per encode call
- UPI Collect API strings (deprecated 28 Feb 2026)
- Runtime branching on the `Mode` string — use `NetworkProfile` fields
- Business logic in `cmd/` packages
- `//nolint` without a comment citing the IC or BUILD reference that permits it

---

## 6. Demo vs Prod Quick Reference

All values come from `NetworkProfile` — never hardcoded.

| Parameter | Demo | Prod |
|---|---|---|
| `DataShards` / `TotalShards` | 3 / 5 | 16 / 56 |
| `ShardSize` | **262144 (identical)** | **262144** |
| `MinDistinctASNs` | **5 (not 2 — MVP §7.1)** | **5** |
| `VettingMinPasses` | 5 | 80 |
| `DepartureThreshold` | 10 min | 72 h |
| `PollingInterval` | 2 min | 24 h |
| `Argon2Time / Memory / Threads` | 1 / 4096 KiB / 1 | 3 / 65536 KiB / 4 |
| `GCRetryBackoff` | 10s, 30s, 2m | 5m, 15m, 60m |
| `ReleaseComputationInterval` | 2 min ticker | 0 (calendar, 23rd) |
| `PaymentMode` | `"mock"` | `"razorpay_live"` |

---

## 7. Session Output

```
✅ Session [ID] — [Title]
Files created:  [paths]
Files modified: [paths]
Tests:          [function names]
VERIFY:         [each check → PASS / actual output]
Engineering:    [any judgment applied in Step 3, or "none"]
TODOs deferred: [list with target session, or "none"]
```

---

## 8. Build Blockers

When a genuine contradiction between governing documents is found:

```
🔴 BUILD BLOCKER — [Short Title]
Conflict:  [doc A says X] vs [doc B says Y]
Affected:  [file or function]
Proposed resolution: [with reasoning]
Action required: confirm / update doc / both
```

Stop immediately. Do not guess or proceed until the user confirms.

---

## 9. Standard session prompt

```
/build
Session [X.Y.Z] — [Title]

Previous: Session [A.B.C] complete.
Files on disk: [e.g. internal/config/network_profile.go, internal/config/profiles.go]
VERIFY from previous: all PASS  (or the checks that failed and what the output was)

[Exact session text from build.md verbatim]

References:
[IC §N.N — paste the section]
[DM §N   — paste the section]
```
---