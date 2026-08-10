# ADR-064 — Demo CLI: all eight subcommands, mock-backed escrow

**Status:** Proposed — blocked on ADR-062
**Track:** DEMO
**Topic:** #14 Client Interface *(new topic)*
**Supersedes:** — *(implements `MVP §8.3`'s `cmd/client` subcommand table)*
**Research source:** N-01 and N-06 above; `MVP §8.3`; `internal/payment/mock.go`;
`internal/client/{account,upload,retrieve,manage}` @ working copy

## Context

N-01: the CLI does not exist and no session builds it. `MVP §8.3` specifies eight subcommands and
maps each to an `internal/client` subpackage; all four subpackages are built and tested. The gap is
purely the entrypoint.

The one genuine design question is scope. `deposit` initiates a UPI Intent deposit, which reads as
production-only — and I initially assumed the demo CLI should omit it. **Checking the code
reversed that** (N-06): upload is gated on escrow balance at `internal/api/upload.go:238`, and
`MockProvider.InitiateEscrow` credits the ledger **synchronously** in demo mode. So `deposit` is not
an optional production nicety — it is a hard prerequisite of the demo's first upload.

### Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Minimal CLI — `upload` and `retrieve` only** | Smallest possible session; matches "it must run" literally | Upload fails immediately without escrow; `register` is required before anything; the demo becomes a scripted fixture rather than a usable tool. Fails your own bar for a hackathon setting |
| **All eight per MVP §8.3 — chosen** | Matches a specification that already exists, so no new design; every subpackage already built and tested; the demo is operable by someone who did not write it | Three subcommands (`ls`, `rm`, `balance`) are not on the critical path — accepted, they are thin wrappers over `internal/client/manage`, already built |
| **Add demo-only convenience commands (auto-fund, one-shot end-to-end)** | Fastest possible live demo | New surface area outside MVP §8.3, in a version that is about to be frozen. A `--seed-escrow` flag on `deposit` gets the same result inside the existing spec |

#### Decision

**1. All eight subcommands per `MVP §8.3`**, each a thin wiring layer over its mapped subpackage.
`cmd/` remains wiring only (IC §11) — any behaviour worth testing belongs in `internal/client`.

**2. `deposit` is demo-critical, not production-only.** In demo mode it calls
`MockProvider.InitiateEscrow`, which credits synchronously. The CLI prints the returned mock VPA and
QR URL, and — per ADR-035 — treats `intent_url` as a **server-owned field it renders, never
constructs.**

**3. Human-readable output; `--json` for the timeline test.** Every subcommand supports `--json` so
`scripts/test/demo_timeline_test.go` can drive the real binary rather than the SDK, which makes the
integration test evidence of *the artifact being demonstrated* rather than of a parallel code path.

**4. All money renders through one formatter.** Paise → rupee conversion happens in exactly one
place, on `int64`, never floating point (IC §11, NFR-038).

**5. Mnemonic display obeys the existing security posture.** `register` prints the BIP-39 mnemonic
once to stdout with a confirmation step, never to a log, never to a file, and never to a
`--json` payload.

**6. Errors render `interface-contracts.md §14` copy codes**, never raw Go error strings — the same
table the future GUI will read.

#### Consequences

The demo becomes demonstrable by a human. The integration test gains the option of driving the real
binary. Cost: one session's worth of wiring over already-tested packages — the cheapest remaining
item on the demo track relative to its value.

**Open constraints:**

- `recover` supports both passphrase and mnemonic paths per `MVP §8.3`. Whether the demo needs both
  is arguable; both are built in `internal/client/account`, so shipping both costs one extra flag.
  Not blocking.
- Progress display for `upload`/`retrieve` (ADR-037's progress/ETA contract) is **out of scope for
  the demo** — a percentage counter is enough. ADR-037's real treatment is LTS. Q-D-4.

---
