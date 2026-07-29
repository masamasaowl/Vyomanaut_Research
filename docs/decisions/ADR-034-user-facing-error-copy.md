# ADR-034 — Canonical User-Facing Copy Layer, Decoupled from the API Error Contract

**Status:** Proposed
**Topic:** #20 Client-Facing UX & Copy *(new topic — no prior ADR owns this surface)*
**Supersedes:** —
**Superseded by:** —
**Research source:** requirements.md §6.1 (Critical UX Moments), §6.2 (Edge States), FR-009; interface-contracts.md §3.3 (Error Envelope Contract); `docs/api/openapi.yaml`; `internal/api/errors.go` and all handler call sites, `Vyomanaut_V2` @ `55467523`

---

## Context

`internal/api/errors.go` implements a machine-readable error envelope (`error_code`, `message`, `request_id`, `retry_after`, `field`, `details`) across **186 `WriteError()` call sites spanning 43 distinct `ErrorCode` values** (owner.go, provider.go, upload.go, otp.go, file.go, token_refresh.go). Every `message` string at every one of those sites is written for an API consumer, not a data owner or provider: `"provider_sig must be 128 lowercase hex characters"`, `"escrow balance insufficient for 30-day storage cost"`, `"missing auth claims"`. That is the correct register for an API layer.

`docs/api/openapi.yaml` independently carries its own example `message` text for many of the same codes, and it is somewhat more complete prose (*"Escrow balance is insufficient to cover 30 days of storage for this file."*, *"Too many OTP requests for this phone number. Try again in 10 minutes."*) — but it remains API-contract documentation: several examples embed a raw UUID or an exact byte-length requirement, appropriate for an integrator and wrong for an end user. So there are, precisely, **two existing technical layers and zero end-user layers** — not one layer with a gap.

Meanwhile `requirements.md` already commits to specific end-user copy in several places that nothing in the codebase implements:

- FR-009: `INSUFFICIENT_ASN_DIVERSITY` must read *"Upload paused — not enough provider diversity. Retry will happen automatically when the network recovers."* The live handler (`internal/api/upload.go`, `writeInsufficientASNDiversityError`) instead emits `"Cannot place %d shards while respecting the per-ASN cap. Current distinct ASNs: %d."` — correct as the API-layer message, but there is no code path anywhere that produces the FR-009 sentence for a human to read.
- §6.1/§6.2 specify plain-language treatments for provider health, degraded files, and escrow holds that likewise have no implementation home.

Two consumers of these 43 error codes are both about to be built from scratch: `cmd/client` (M15, data-owner-facing) and `cmd/provider` (M13, provider-daemon-facing) — both currently 155–157-byte stub `main.go` files with no orchestration logic. Four codes are already confirmed to surface through **both** future apps (`ErrInternal`, `ErrInvalidRequest`, `ErrPhoneAlreadyRegistered`, `ErrUnauthorized` all appear in both `owner.go` and `provider.go` today), and that overlap will only grow as M13/M15 are built out. If each is built independently against the raw `error_code` values with copy invented ad hoc per call site, the two apps will diverge in tone, terminology, and completeness for the same underlying condition — and any future third consumer (a GUI, per the `provider_ui_enabled` flag already reserved in `mvp.md`) would repeat the divergence a third time.

## Decision

Introduce a single canonical **copy table** mapping `error_code → {headline, body, suggested_action, severity}`, consumed by every user-facing surface, and kept explicitly separate from the OAS `message` field.

### 1. Location and shape

A new file, `docs/system-design/ux-copy.md`, holds the canonical table — one row per `ErrorCode` constant in `internal/api/errors.go`, e.g.:

| `error_code` | Headline | Body | Suggested action | Severity |
| --- | --- | --- | --- | --- |
| `INSUFFICIENT_ASN_DIVERSITY` | Upload paused | Not enough provider diversity right now. | Retrying automatically — no action needed. | info / auto-retry |
| `ESCROW_INSUFFICIENT` | Add funds to continue | Your balance won't cover 30 days of storage for this file. | Top up your balance to resume. | action-required |
| `PROVIDER_DEPARTED` | This provider has left the network | — | No action needed; repair will replace it automatically. | info |

This is documentation, not code, deliberately — it stays reviewable by non-engineers (matches the discipline already used for `docs/system-design/*.md`) and is trivially diffable in PRs.

### 2. Consumption contract

`cmd/client` and `cmd/provider` both look up `error_code` (never `message`) against this table at render time. `message` remains for logs, support escalation, and `--verbose`/debug output — never shown as the primary line in normal operation.

### 3. Fallback for missing entries

Any `error_code` without a table entry renders a generic fallback: *"Something went wrong (code: {error_code}). Try again, or contact support with this code."* — logged as a warning client-side so gaps get caught in testing rather than silently shipping a raw API string to a user. This means the table can be built out incrementally without blocking M13/M15 on 100% coverage on day one.

### 4. Process going forward

Adding a new `ErrorCode` to `internal/api/errors.go` should add a corresponding row to `ux-copy.md` in the same change — the same reconciliation discipline `errors.go`'s own header comment already applies to keeping Go and the OAS in sync, extended to a third artifact.

## Alternatives considered

- **A — Each app writes its own copy as it's built (status quo / do nothing).** Rejected: four codes are already provably shared across both future consumers; independent invention guarantees drift, and the cost of reconciling two already-shipped apps later is far higher than designing one table now, before either exists.
- **B — Extend the OAS `message` field itself to be the human-facing copy.** Rejected: the OAS message is API-contract documentation, potentially consumed by third-party integrators and admin tooling, evidenced by its own examples still carrying raw UUIDs and exact byte-length specs. Overloading it with a second, user-tone responsibility would make the API contract unstable to copywriting iteration and blur what the contract actually promises.
- **C — A Go package (`internal/uxcopy`) instead of a markdown table.** Considered viable but deferred: a markdown table is reviewable by whoever owns tone/voice without a Go toolchain, and can be mechanically compiled into a Go map or JSON asset at build time if `cmd/client`/`cmd/provider` end up wanting compile-time safety. Worth revisiting once M15 is underway if hand-authored maps prove error-prone.

## Consequences

**Positive:** M13 and M15 both consume one source of truth instead of inventing copy independently; the FR-009-style commitments already made in `requirements.md` finally have an implementation path; new error codes have a defined process rather than an ad hoc one.

**Negative / trade-offs:** a new artifact to maintain and keep in sync as `errors.go` evolves; initial build-out is nontrivial copywriting work across 43 codes (mitigated by the fallback in §3, so it can ship incomplete and be filled in).

**Affected:** no changes to already-shipped code (`internal/api`, M11, is untouched — this is additive at the client/daemon layer only). New artifact: `docs/system-design/ux-copy.md`. Downstream: `cmd/client` (M15), `cmd/provider` (M13) must consume it rather than hard-coding strings.

## Validation (once M13/M15 exist)

1. Every `error_code` actually returned by the live API (43 today) either has a table entry or exercises the defined fallback — no raw API `message` string should ever reach a terminal in normal operation.
2. Comprehension-test the highest-frequency/highest-stakes entries (`ESCROW_INSUFFICIENT`, `INSUFFICIENT_ASN_DIVERSITY`, `PROVIDER_DEPARTED`, `RAZORPAY_UNAVAILABLE`) with representative data owners and providers before general release, per the earlier UX research recommendation.
