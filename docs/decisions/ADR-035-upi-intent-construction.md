# ADR-035 — UPI Intent Link Construction Is Server-Owned, Not Client-Derived

**Status:** Proposed
**Topic:** #13 Escrow & Payment Basis (deposit flow)
**Supersedes:** —
**Superseded by:** —
**Research source:** requirements.md FR-006, FR-014; `docs/api/openapi.yaml` `DepositInitiateResponse`; ADR-011 (escrow-payments, Smart Collect VPA precedent); `internal/payment/razorpay.go`, `internal/payment/mock.go`, `Vyomanaut_V2` @ `55467523`

---

## Context

`requirements.md` FR-006/FR-014 require the client to *"surface a UPI Intent deposit link pre-populated with the shortfall amount."* The live `DepositInitiateResponse` (`docs/api/openapi.yaml`, backed by `OwnerDepositHandler` in `internal/api/owner.go`) returns exactly two fields: `vpa` and `qr_code_url`. The OAS's own description of `qr_code_url` is *"URL of a QR code image encoding the UPI Intent deep link"* — confirming the deep link exists, but only encoded inside an image, never exposed as a plain string.

This gap runs all the way down the stack, not just at the HTTP boundary: `RazorpayClient.CreateVirtualAccount` (`internal/payment/razorpay.go`) is declared to return `(vpa string, qrURL string, err error)` — the interface itself has no slot for an intent URI. `internal/payment/mock.go`'s demo-mode implementation mirrors the same two-value shape. There is no `upi://pay?...` string constructed, stored, or returned anywhere in `internal/payment` or `internal/api` (confirmed by search — no `upi://`, `intent_url`, or equivalent identifier exists in the current codebase).

This matters beyond copy because V2's data-owner client (`cmd/client`, M15) is CLI-only — there is no planned GUI for data owners, and providers' own tray/GUI (`provider_ui_enabled`) is separately flagged off by default. A QR *image* is awkward for a CLI: a data owner running `cmd/client deposit` from a terminal — including headless/server scenarios, which the CLI-only design implies as a real usage mode — has no convenient way to view an image URL at all, let alone scan it with a phone camera pointed at a screen. FR-014's "deposit link" language reads as expecting something a CLI can print and a user can tap or paste, not an image.

The correctness stakes are real, not cosmetic: a malformed UPI intent URI is a hard payment failure, and it would surface at the single highest-friction moment in the data-owner journey — topping up money before anything else in the product can be used. If the client independently re-derives the URI from the bare `vpa` (guessing at parameter names, amount encoding, transaction-reference format), a bug in that encoding is a client-side defect in a domain (payment parameter construction) the client has no other reason to own.

## Decision

The **payment layer constructs and returns the UPI intent URI as a plain string**; the client only renders it. The QR image remains, unchanged, as a secondary convenience.

### 1. New response field

`DepositInitiateResponse` gains `intent_url` (string), alongside the existing `vpa`, `qr_code_url`, `expires_at`:

```yaml
DepositInitiateResponse:
  required: [vpa, qr_code_url, intent_url]
  properties:
    vpa: {type: string}
    qr_code_url: {type: string}
    intent_url:
      type: string
      description: >
        Fully-formed UPI Intent deep link (upi://pay?...), pre-populated with
        this deposit's VPA, amount, and transaction reference. The client
        renders this directly; it must not re-derive it from vpa.
    expires_at: {type: string, format: date-time}
```

### 2. Where it's built

`internal/payment`'s `CreateVirtualAccount` (or a thin wrapper called immediately after it, so neither `RazorpayProvider` nor `MockProvider` has to duplicate the encoding) constructs:

```
upi://pay?pa={vpa}&pn=Vyomanaut&am={amount_rupees}&cu=INR&tr={contract_id}
```

using the same `amountPaise`/`contractID` already passed into `CreateVirtualAccount` today — no new inputs are required, only a new deterministic output alongside the existing `vpa`/`qrURL` pair. `RazorpayClient.CreateVirtualAccount`'s signature extends to `(vpa string, qrURL string, intentURL string, err error)` (or an equivalent struct return, at the implementer's discretion), so both `RazorpayProvider` and `MockProvider` are forced to produce it identically in shape.

### 3. Client behavior

`cmd/client deposit` prints `intent_url` as the primary, copyable/tappable output (most modern terminals render `OSC 8` hyperlinks; a plain string is still directly usable via copy-paste into a UPI app). The QR image URL remains available for a future GUI/mobile surface or a second-device scan, but is not the CLI's primary path.

## Alternatives considered

- **A — Client constructs the UPI URI itself from `vpa` + amount already returned.** Rejected: duplicates payment-critical parameter encoding outside the package that owns payments, and the moment a second consumer exists (a future GUI, per the `provider_ui_enabled` precedent already reserved in `mvp.md`) the encoding logic would need to be kept in sync in two places — a single malformed-URI bug would need to be found and fixed twice.
- **B — Ship only the QR image; tell the CLI user to scan it with their phone.** Rejected: blocks the stated CLI-only, headless-capable usage mode outright — there may be no convenient way to view an image URL at all in that scenario, let alone scan it.
- **C — Server returns `intent_url` as a plain string; QR stays as a secondary convenience.** Adopted.

## Consequences

**Positive:** payment-parameter encoding stays owned by the package that owns payments; the CLI gets a directly usable deposit path without inventing URI construction; a future GUI consumer gets the same field for free.

**Negative / trade-offs:** a small, additive change to an already-built, already-shipped package (`internal/payment`, M10) and its OAS schema — low risk, but M10 is technically "done" and this reopens it slightly.

**Affected:** `internal/payment/razorpay.go`, `internal/payment/mock.go` (interface signature + implementations), `internal/api/owner.go` (`OwnerDepositHandler` response body), `docs/api/openapi.yaml` (`DepositInitiateResponse` schema). No schema/ledger/RLS change — `intent_url` is derived at request time from data already in hand, not persisted.

## Validation (once implemented)

1. Construct `intent_url` with a known `vpa`/`amount`/`contract_id` and confirm it parses correctly against the UPI deep-link grammar (`pa`, `pn`, `am`, `cu`, `tr` keys present and correctly encoded) before M15 ships.
2. Confirm `mock.go` and `razorpay.go` produce structurally identical `intent_url` output (only the VPA domain differs) so demo mode remains a faithful preview of production behavior — consistent with the demo/production parity principle already established by ADR-031.
