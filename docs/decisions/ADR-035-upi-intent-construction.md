# ADR-035 — UPI Intent Link Construction Is Server-Owned, Not Client-Derived

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (deposit flow)
**Supersedes:** —
**Superseded by:** ADR-038 — context only (the premise that `cmd/client` has no planned GUI). The intent_url decision, schema, and construction logic below are not superseded — see the corrected Context and Client Behavior text below.
**Research source:** requirements.md FR-006, FR-014; `docs/api/openapi.yaml` `DepositInitiateResponse`; ADR-011 (escrow-payments, Smart Collect VPA precedent); `internal/payment/razorpay.go`, `internal/payment/mock.go`, `Vyomanaut_V2` @ `55467523`

---

## Context

`requirements.md` FR-006/FR-014 require the client to *"surface a UPI Intent deposit link pre-populated with the shortfall amount."* The live `DepositInitiateResponse` (`docs/api/openapi.yaml`, backed by `OwnerDepositHandler` in `internal/api/owner.go`) returns exactly two fields: `vpa` and `qr_code_url`. The OAS's own description of `qr_code_url` is *"URL of a QR code image encoding the UPI Intent deep link"* — confirming the deep link exists, but only encoded inside an image, never exposed as a plain string.

This gap runs all the way down the stack, not just at the HTTP boundary: `RazorpayClient.CreateVirtualAccount` (`internal/payment/razorpay.go`) is declared to return `(vpa string, qrURL string, err error)` — the interface itself has no slot for an intent URI. `internal/payment/mock.go`'s demo-mode implementation mirrors the same two-value shape. There is no `upi://pay?...` string constructed, stored, or returned anywhere in `internal/payment` or `internal/api` (confirmed by search — no `upi://`, `intent_url`, or equivalent identifier exists in the current codebase).

This matters beyond copy because V2 ships a Wails-based Data Owner GUI app (ADR-038) as the primary interface, with `cmd/client` retained underneath for power users and headless/scripted use — not the other way around, as earlier drafts of this ADR assumed. Both interfaces need `intent_url`: the GUI can render it as a tappable link (or inline QR, since a webview can display an image directly) while a CLI/headless session — still a real, supported usage mode — has no convenient way to view an image URL at all, let alone scan it. Either way, the underlying gap is unchanged: nothing in `internal/payment` or `internal/api` currently constructs or returns a plain-string deep link, and both interfaces need one.

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

Both consumers render the same `intent_url` field, appropriate to their surface: the Wails Data Owner app (ADR-038) renders it as a tappable link and may additionally show the QR image inline, since a webview can display both natively. `cmd/client deposit` — retained for power users and headless/scripted use — prints `intent_url` as its primary, copyable output (most modern terminals render `OSC 8` hyperlinks; a plain string is still directly usable via copy-paste into a UPI app). Neither interface re-derives the URI itself; both consume the same server-constructed field.
## Alternatives considered

- **A — Client constructs the UPI URI itself from `vpa` + amount already returned.** Rejected: duplicates payment-critical parameter encoding outside the package that owns payments — and with both the Wails GUI (ADR-038) and `cmd/client` as real, concurrent consumers of this field, not one future hypothetical one, a single malformed-URI bug would need to be found and fixed in two places instead of one.
- **B — Ship only the QR image; tell the CLI user to scan it with their phone.** Rejected: blocks the stated CLI-only, headless-capable usage mode outright — there may be no convenient way to view an image URL at all in that scenario, let alone scan it.
- **C — Server returns `intent_url` as a plain string; QR stays as a secondary convenience.** Adopted.

## Consequences

**Positive:** payment-parameter encoding stays owned by the package that owns payments; the CLI gets a directly usable deposit path without inventing URI construction; a future GUI consumer gets the same field for free.

**Negative / trade-offs:** a small, additive change to an already-built, already-shipped package (`internal/payment`, M10) and its OAS schema — low risk, but M10 is technically "done" and this reopens it slightly.

**Affected:** `internal/payment/razorpay.go`, `internal/payment/mock.go` (interface signature + implementations), `internal/api/owner.go` (`OwnerDepositHandler` response body), `docs/api/openapi.yaml` (`DepositInitiateResponse` schema). No schema/ledger/RLS change — `intent_url` is derived at request time from data already in hand, not persisted.

## Validation (once implemented)

1. Construct `intent_url` with a known `vpa`/`amount`/`contract_id` and confirm it parses correctly against the UPI deep-link grammar (`pa`, `pn`, `am`, `cu`, `tr` keys present and correctly encoded) before M15 ships.
2. Confirm `mock.go` and `razorpay.go` produce structurally identical `intent_url` output (only the VPA domain differs) so demo mode remains a faithful preview of production behavior — consistent with the demo/production parity principle already established by ADR-031.
