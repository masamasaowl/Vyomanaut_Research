---
name: build
description: >
  Executes a build session for the Vyomanaut V2 distributed storage system.
  Trigger this skill when the user provides a Build Instruction (format: "Build
  Instruction #N: [description]") or says "continue the build", "next milestone",
  "implement M[N]", or "start coding". Also trigger when the user provides code
  context and asks to generate, extend, or fix Go code for the Vyomanaut project.
  This skill enforces the mandatory pre-read protocol, milestone tracing, invariant
  safety, and session handoff format. Never generate code for this project without
  this skill active.
---

# Build Skill — Vyomanaut V2

Produces session-scoped, production-ready Go code for the Vyomanaut V2 distributed
storage system. Every artifact traces to a milestone, satisfies FRs/NFRs, and upholds
the five data-model invariants from `data-model.md §3`.

---

## Reference files — read before anything else

| File | When to read | What it gates |
|---|---|---|
| `docs/system-design/mvp.md` | Every session, mandatory | Milestone scope, exit criteria, dependency graph |
| `coding-guide.md` | Every session, mandatory | All code generation rules — this is the authority on scope sizing |
| `docs/system-design/data-model.md §3` | Every session, mandatory | Five invariants code must not violate |
| `docs/system-design/interface-contracts.md §5` | When implementing a package | Pre-defined Go function signatures — match exactly |
| `testing-strategy.md` | When generating tests | Test names, assertions, CI tag requirements |
| `docs/system-design/requirements.md` | When an FR/NFR is ambiguous | Authoritative requirement text |
| `docs/decisions/ADR-NNN-*.md` | When a design decision is in scope | ADR wins over all other guidance |

**Authority chain** (from `docs/system-design/README.md §0.2`):
```
ADR
  > requirements.md
    > architecture.md / data-model.md / interface-contracts.md
      > coding-guide.md
        > testing-strategy.md
          > general Go convention
```
A conflict between any two sources must be flagged and resolved. Never resolve silently.

---

## Step 0 — Context Gate (unconditional, runs first)

Before any other step, verify:

1. **Build Instruction number is present.** If absent: ask, do not proceed.
2. **Context window contains all mandatory reference files listed above.** If any are
   absent: name them and output `CONTEXT GATE FAIL — missing: [file list]`. Do not
   generate code with a stale or incomplete context.
3. **The previous session's Output B is in context** (if this is not session #1).
   The handoff from Output B is the continuity mechanism — it carries the
   `BUILD-TODO` markers that define this session's starting state. If session #1: skip.

> If any check fails, output exactly this and stop:
> `CONTEXT GATE FAIL — [reason]. Please provide: [specific missing items].`

---

## Step 1 — Milestone Trace

Map the Build Instruction to a milestone in `docs/system-design/mvp.md`.

Output this block before writing any code:

```
Milestone: M[N] — [Milestone name]
Sprint: [N]
Depends on: [prior milestones that must be complete]
FR/NFR satisfied by this session: [list with IDs]
Invariants in scope: [which of 1–5 from data-model.md §3]
Files targeted this session: [specific files from coding-guide.md §Package Layout]
Exit criterion check: [exact command from mvp.md that proves this session's work]
```

**For milestones spanning multiple sessions** (e.g. M4 touches 4+ files):
- Identify which file is the *entry point* — the one whose interface all others depend on.
  For M4: `internal/audit/secret.go` (cluster secret) → `internal/audit/challenge.go`
  (nonce generation) → `internal/audit/receipt.go` (two-phase write) → `internal/audit/jit.go`.
- Produce the entry-point file in full and scaffold the rest with `BUILD-TODO` markers.
- Output B of this session becomes "Build Instruction #N+1: implement [next file]".

If the Build Instruction cannot be traced to a milestone: **pause and request clarification.**
Do not invent a scope.

---

## Step 2 — Pre-Code Checklist

Run this checklist before writing the first function. Each failed item must be corrected
before proceeding — do not generate code and annotate it as "to be fixed later".

**Cryptographic safety:**
- [ ] AONT key K never appears in any log statement, error string, struct field, or
      return value. K is generated, used, and zeroed inside the encode function only.
- [ ] Poly1305 and any other auth-tag comparison uses `crypto/subtle.ConstantTimeCompare`.
      `bytes.Equal` is prohibited for auth-tag paths. No exceptions.
- [ ] Pointer file nonce counter is incremented and persisted BEFORE the AEAD call.
      A crash between encrypt and persist would reuse the nonce; nonce reuse destroys AEAD.
- [ ] `challenge_nonce` is typed as `[33]byte` (or `ChallengeNonce` type alias) — never `[32]byte`.

**Code contracts:**
- [ ] Every exported function signature matches `interface-contracts.md §5` exactly
      if that function is pre-defined there. Parameter names, types, and order are binding.
- [ ] Every exported function that does I/O takes `context.Context` as its first argument.
- [ ] No `float64` or `float32` anywhere in `internal/payment/` or any file it imports.
- [ ] No import relationships that violate the dependency chain in `coding-guide.md §2`.

**Tests:**
- [ ] Every new integration test file carries `//go:build integration` as its first non-blank line.
- [ ] Every test function includes `t.Parallel()` unless it uses a named shared resource
      that explicitly prohibits concurrent access (and that reason must be in a comment).

**Database:**
- [ ] No raw SQL string concatenation. Parameterised queries only.
- [ ] Two-phase audit receipt writes use two SEPARATE database calls (NOT one transaction).
      Phase 1 is a standalone `db.ExecContext` INSERT. Phase 2 is a standalone `db.ExecContext`
      UPDATE. They are deliberately not in a transaction — the crash-safety property
      depends on Phase 1 being durable before Phase 2 begins. See `architecture.md §14`.

---

## Step 3 — Session Scope

**Scope sizing is governed by `coding-guide.md §12`.** That section is the single authority.
Do not apply a different scope rule. Key points:

- M0–M2: 1–3 files per session, full content. These packages are small and correctness-critical.
- M3–M5: 1–2 files per session. Complex files get scaffold with `BUILD-TODO` markers.
- M6–M8: 1 file per session. Driven strictly by `interface-contracts.md`.

**BUILD-TODO marker format** (from `coding-guide.md §12`):
```go
// BUILD-TODO(M[milestone]-S[session]): [what to implement]
// Requires: [ADR, paper, or interface-contracts.md §reference]
func (h *Host) maintainRelayReservation(ctx context.Context) error {
    panic("not implemented — BUILD-TODO(M3-S2)")
}
```
The `panic` ensures tests fail loudly if this function is called. Never use
`return nil` as a stub for a security-critical function.

---

## Step 4 — Generate Outputs

Produce the outputs listed below, in this order. Do not add others.

---

### Output A — New Code

One or more production-ready Go files.

**File header format (mandatory):**
```go
// Package [name] implements [one-sentence description].
//
// Milestone: M[N] — [Milestone name]
// Satisfies: [FR-NNN, NFR-NNN list]
// ADRs: [ADR-NNN list]
// Authority: interface-contracts.md §[N.N] (if signatures are pre-defined)
package [name]
```

**Every exported function doc comment must cite its FR/NFR:**
```go
// AONTEncodeSegment applies the All-or-Nothing Transform to a plaintext segment.
// The AONT key K is generated fresh, embedded in the output, and zeroed on return.
// K never leaves this function.
//
// Satisfies: FR-007, FR-008 (ADR-019, ADR-022)
func AONTEncodeSegment(segment []byte, aesNIAvailable bool) ([]byte, error) {
```

**Compile-time interface assertion** — required for every concrete type implementing
an exported interface:
```go
// Compile-time interface check — fails at build time if Host does not implement p2p.Host.
var _ p2p.Host = (*host)(nil)
```

**Rules:**
- Runs `go build`, `go vet`, passes `-race` conceptually (no data races in design).
- No `//nolint` directives without a comment explaining why, approved by two engineers.
- BUILD-TODO functions use `panic("not implemented — BUILD-TODO(MN-SN)")`, never `return nil`.

---

### Output B — Build Instruction for Next Session

```
Build Instruction #[N+1]: [One-sentence description of exactly what to implement]

Milestone: M[N] (continued) OR M[N+1] — [Milestone name]
File to produce: [specific filename from coding-guide.md §Package Layout]

Source material required in context:
  - docs/system-design/mvp.md (always)
  - coding-guide.md (always)
  - testing-strategy.md (if tests are generated)
  - docs/system-design/interface-contracts.md §[specific section]
  - docs/decisions/ADR-NNN-*.md (list relevant ADRs)
  - [paper files if a cryptographic or erasure-coding decision must be verified]

BUILD-TODOs carrying over from this session:
  - [BUILD-TODO(MN-SN) text] in [filename] → this session must implement this
  - [or "none — all functions implemented"]

Benchmark required before this session closes: [YES/NO]
  If YES: [NFR-NNN target, Q-number from benchmarking-protocol.md, hardware spec required]
```

If this session completes a milestone's exit criterion: note it explicitly and set the
next Build Instruction to the first file of the next milestone.

---

### Output C — Bug Report

Produce this section only if bugs were identified during this session.

**Severity classification:**

| Severity | Definition | Required action |
|---|---|---|
| **CRITICAL** | Violates a data-model invariant, leaks crypto key material, produces wrong money amounts, or breaks crash safety | HALT build. Fix before any new code. Note which invariant is broken. |
| **HIGH** | Incorrect behaviour in a critical path that tests would catch | Fix in this session or block next session until fixed |
| **MEDIUM** | Logic error outside critical paths | Note in Output C; fix in next appropriate session |
| **LOW** | Style, documentation, or non-functional | Note only; fix opportunistically |

**Format per bug:**
```
BUG-[N] [SEVERITY]
Location: [package/file:line or function name]
Invariant violated: [1–5 from data-model.md §3, or "none"]
Description: [what is wrong and why]
Fix: [exact change required — code snippet if possible]
Test that would catch this: [test name from testing-strategy.md]
```

---

### Output D — Tests

Produce this only when the session generates testable code.

Rules:
- Test names must match `testing-strategy.md` exactly. Do not invent new names for
  functions that already have a defined test name.
- Table-driven subtests for every function under test.
- `t.Parallel()` on every test function unless a comment explains why it cannot run concurrently.
- Integration tests: `//go:build integration` on line 1 of the file, before the package declaration.
- Fuzz targets: include a `testdata/fuzz/[FuzzFunctionName]/` subdirectory with at
  minimum these seed inputs: zero-length, minimum valid, minimum valid with one bit
  flipped, maximum valid.
- Benchmarks: include the hardware context comment per `testing-strategy.md §5`:
  ```go
  // Hardware: [CPU, RAM, disk type]
  // Target: NFR-[N] — [target value]
  func BenchmarkAONTEncode14MB(b *testing.B) {
  ```

If a function in Output A has no corresponding test in `testing-strategy.md`: name the gap
and add the missing test name to Output B for the next session.

---

### Output E — Session Record

```
Session #[N] complete.
Milestone: M[N] — [Milestone name]

Exit criterion status: [one of: PASSING / PARTIAL / BLOCKED]
  (PASSING = exact mvp.md command would succeed)
  (PARTIAL = some but not all deliverables complete; BUILD-TODOs remain)
  (BLOCKED = a dependency or decision is unresolved; must be resolved before next session)

FR/NFR satisfied this session:
  ✓ FR-NNN — [description]
  ✓ NFR-NNN — [description]
  — FR-NNN — [description] [deferred — BUILD-TODO(MN-SN)]

Invariants upheld:
  Invariant [N]: [how this session's code maintains it — specific mechanism, not assertion]

Benchmark results (if any):
  [NFR-NNN] [benchmark name]: p50=[N]ms, p99=[N]ms on [hardware spec]
  Pass/Fail vs target: [PASS/FAIL]

Files produced:
  [filename] — [one-sentence description of what it implements]

Next: Build Instruction #[N+1]
  [copy of Output B]
```

---

## Pause Conditions

Stop code generation immediately and output `BUILD PAUSED — [reason]`:

**P1 — Logical contradiction requiring research:**
```
BUILD PAUSED — Logical error in [package/decision].
Requires: [specific paper-NN or ADR-NNN].
If not in context: request access to Vyomanaut Research repository at
https://github.com/masamasaowl/Vyomanaut_Research before resuming.
```

**P2 — Required reference absent from context:**
```
BUILD PAUSED — [ADR-NNN or paper-NN] required for [specific decision].
Add to context and restart Session #[N].
```

**P3 — Critical design decision with multiple valid options:**
```
BUILD PAUSED — Decision required: [question].
Option A: [description] — consequence: [X]
Option B: [description] — consequence: [Y]
Awaiting developer input before proceeding.
```

**P4 — Context gate failed (Step 0):**
```
CONTEXT GATE FAIL — [reason]. Please provide: [specific missing items].
```

**P5 — Benchmark target likely missed:**
```
BUILD PAUSED — NFR-[N] target ([target value]) may not be achievable with this
implementation approach. Run benchmark [name] from testing-strategy.md §5 on
minimum-spec hardware (dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD) per
benchmarking-protocol.md Q[N]-[N] before proceeding.
```

**P6 — Invariant violation detected in new code:**
```
BUILD PAUSED — Invariant [N] violation in [function/file].
[Exact description of the violation.]
This must be fixed before any further code is written this session.
Fix: [exact change required]
```

Do not generate code after a PAUSE output. Wait for developer input.

---

## Source Precedence Reference

When two sources conflict:

```
ADR (Accepted)
  > requirements.md FR/NFR
    > architecture.md
      > data-model.md §3 invariants
        > interface-contracts.md §5 signatures
          > coding-guide.md patterns
            > testing-strategy.md test names
              > general Go 1.22+ convention
```

Flag every conflict explicitly. Never resolve silently by choosing the more convenient source.

---

## Repository Note

The Vyomanaut Research repository at https://github.com/masamasaowl/Vyomanaut_Research
is private. If a research paper (e.g. `docs/research/paper-16-aont-rs-dispersal.md`)
is needed to verify a cryptographic or erasure-coding design choice and is not in
context: output a P1 PAUSE with an explicit repository access request.

Do not proceed from memory on any of: AONT construction, Reed-Solomon parameters,
HKDF derivation strings, Argon2id parameters, or Razorpay API behaviour.
These are exactly the decisions where memory errors produce silent, undetectable bugs.