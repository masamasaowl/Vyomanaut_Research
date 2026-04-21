# Paper 35 — Razorpay API Documentation and UPI Payment Technical Reference

**Authors:** Razorpay / NPCI (National Payments Corporation of India)
**Venue / Year:** Razorpay Developer Docs (docs.razorpay.com) | Continuously maintained; retrieved April 2026
**Topics:** #13, #18
**ADRs produced:** none — ADR-011 gains concrete implementation constraints; ADR-024 gains one critical constraint (escrow hold architecture must be custom ledger, not native Razorpay Escrow); see Decisions Influenced

---

## Problem Solved

ADR-011 (fiat escrow via Razorpay/UPI) and ADR-024 (graduated partial-release escrow) assumed that Razorpay provides a native partial-hold-and-release escrow primitive. This documentation review tests that assumption against the actual API surface and finds it false in an important way: Razorpay's Escrow+ product is a regulated tri-party bank account for lending/P2P NBFC use cases, not a programmable held-earnings store. The practical escrow mechanism for Vyomanaut is Razorpay Route (split payment with settlement hold), not Razorpay Escrow+. A second critical finding is that NPCI is deprecating UPI Collect flow effective 28 February 2026, requiring a migration to UPI Intent or QR code for provider-side deposits. This paper maps the full API surface relevant to ADR-011 and ADR-024, identifies the correct product for each payment flow, and surfaces the constraints that change the implementation plan.

---

## Key Findings

**The two Razorpay products relevant to Vyomanaut are distinct (RazorpayX vs Razorpay Route):**

RazorpayX is the business banking suite: current accounts, payouts (IMPS/NEFT/RTGS/UPI) to external bank accounts, and the Escrow+ product. Razorpay Route is the payment-splitting product on the Payment Gateway side: it lets a merchant receive a payment and then transfer portions to Linked Accounts with programmable settlement schedules. These are separate API stacks with different authentication, different base URLs, and different trust models.

**Razorpay Escrow+ is not the right product for Vyomanaut (Escrow docs):**
RazorpayX Escrow+ is a regulated bank account product for P2P lending NBFCs and co-lending companies. It requires a registered trustee (Axis Trustee or Mitcon Credentia), a signed tri-party Escrow Agreement between Razorpay, the Escrow Bank, and the Trustee, and NBFC registration. Onboarding is manual: the business contacts a Razorpay POC, who arranges the Trustee introduction and bank account opening. The only documented use cases are P2P lending and co-lending. Payout purposes are restricted to "Loan disbursement" and "Investor withdrawal." Vyomanaut is not an NBFC. The Escrow+ product is inaccessible for a general storage marketplace.

**Razorpay Route is the correct escrow substitute — and it has the hold primitive ADR-024 needs (Route docs, Transfer API):**
Route allows a merchant to receive a payment and create one or more transfers to Linked Accounts. Each transfer carries two settlement control parameters: `on_hold` (boolean) and `on_hold_until` (Unix timestamp). Setting `on_hold: true` with no timestamp holds the settlement indefinitely. Setting `on_hold_until` to a future Unix timestamp holds it until that date and releases automatically on the next business day. The Modify Settlement Hold API (`PATCH /v1/transfers/:transfer_id`) lets the microservice update the hold state at any time — the exact "partial earnings release on first of month" behaviour ADR-024 requires. This is a standard API, not an on-demand feature; it is available on all Route accounts.

**Route Linked Accounts — the provider account model:**
Each Vyomanaut provider must be onboarded as a Linked Account on the Razorpay Route dashboard. A Linked Account is a registered business or individual with a Razorpay account. After a 24-hour cooling period, the microservice can begin creating transfers to the Linked Account. Linked Accounts have their own settlement schedule (inherits the master account's T+N schedule by default, adjustable by contacting Razorpay support). Linked Account holders can access their own dashboard to view transactions. Route supports three transfer initiation methods: via Orders (created at order time, executed on payment capture), via Payments (triggered after payment capture), and via Direct Transfer (from the master account balance, on-demand feature requiring support activation).

**Payout API for provider earnings release (RazorpayX Payouts docs):**
When the microservice executes a monthly escrow release, the flow is: the master Razorpay account holds the accumulated earned-but-unreleased balance. The microservice calls the Payout API to send the released fraction to the provider's registered bank account (UPI VPA or bank account number). Payout modes: IMPS (₹0–₹5 lakh per transaction, 24x7, instant), NEFT (batched, bank hours), RTGS (₹2 lakh minimum, bank hours), UPI (24x7, instant for amounts within UPI limits). For provider monthly payouts, IMPS or UPI is the correct mode.

**Idempotency key is mandatory for all payouts as of 15 March 2025 (Payout Best Practices docs):**
The header `X-Payout-Idempotency` must be passed with every payout API call. The key must be unique per intended transaction; if retried after a network failure, the same key must be reused (the system deduplicates and returns the existing payout state). This is already required by ADR-016 (idempotency key on escrow_events rows) but now has a hard external enforcement deadline: payouts without an idempotency key are rejected by Razorpay. The escrow_events `idempotency_key` field (ADR-016) should be used directly as the `X-Payout-Idempotency` header value.

**Payout life cycle — states relevant to ADR-024 (Payout States docs):**
A payout can be in: `pending` (awaiting internal approval if Approval Workflow is enabled), `queued` (insufficient balance), `scheduled` (future-dated), `processing` (submitted to bank), `processed` (terminal success), `reversed` (terminal failure, balance refunded), `cancelled`, `rejected`, `failed`. For Vyomanaut's monthly release, the relevant path is: payout created → `processing` → `processed`. If the payout fails (`reversed`), Razorpay automatically credits the amount back to the master account — no manual recovery step needed for the payment layer. The microservice must handle webhook events `payout.processed` and `payout.reversed` to update the escrow ledger accordingly.

**UPI Collect flow is deprecated — migration to UPI Intent is required by 28 February 2026 (UPI docs):**
NPCI is deprecating UPI Collect (where a payer enters their VPA manually to initiate a payment) effective 28 February 2026. The only exempted MCCs are 6012 and 6211 (IPO and secondary market transactions). Vyomanaut data owners depositing escrow funds via UPI currently use the Collect flow (they enter their UPI ID in the checkout). This flow will stop working at the deprecation date. The replacement is UPI Intent (the payer selects their UPI app from a displayed list, the app is launched automatically, payment details are prefilled) or UPI QR code. For the V2 launch, UPI Intent must be the primary deposit method. Smart Collect 2.0 (which creates virtual UPI IDs backed by a current or escrow account) is unaffected — it operates on the receiving side, not the payer side.

**Smart Collect 2.0 for programmatic UPI collection (Smart Collect docs):**
Smart Collect 2.0 creates Customer Identifiers (virtual bank accounts and virtual UPI IDs) linked to the Razorpay master account. Each data owner can be assigned a unique Customer Identifier with a VPA; they send escrow deposits to that VPA, and Razorpay automates reconciliation and notifies via webhook. This is the correct mechanism for data owner deposit collection — it maps each upload contract to a unique virtual account, making reconciliation automatic. Smart Collect 2.0 requires a Current Account or Escrow+ account (Escrow+ for regulatory reasons if using bank account receiver; Current Account is sufficient for UPI VPA receiver).

**Payout fee structure (Payout docs):**
IMPS payouts: fees vary by bank partnership (typically ₹2.5–₹5 per transaction for amounts up to ₹5 lakh). UPI payouts to VPA: typically ₹0 for UPI transfer within UPI rails. The exact fee is returned in the payout response (`fees` and `tax` fields in paise). All fees are deducted from the master account balance, not from the payout amount. This means the gross payout to the provider is exactly the amount specified — fees do not reduce the provider's payment.

**Route transfer reversal for seizure events (Route Reversal API):**
Route provides a reversal API that recalls funds from a Linked Account back to the master account. This is designed for customer refunds but is equally usable for escrow seizure events (provider departed silently, funds must be recalled from the Route settlement and moved to the repair reserve fund). If a transfer has already settled to the Linked Account, the reversal debits the Linked Account and credits the master account. If the Linked Account has insufficient balance, the reversal fails — the seizure mechanism must therefore be triggered before settlement by keeping earnings on hold until the departure threshold resolves.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Razorpay Route (split payment + hold) as escrow primitive | RazorpayX Escrow+ (regulated tri-party account) | Full programmatic control over hold/release/seizure via API; no NBFC requirement; no trustee overhead; not a "real" escrow account from a legal standpoint |
| Custom internal escrow ledger (escrow_events table, ADR-016) + Razorpay Route as the payout rail | Razorpay native partial-release escrow | Complete control over hold logic, multipliers, and seizure; Razorpay is used only for the final fiat transfer; adds implementation complexity |
| IMPS/UPI for provider payouts | NEFT/RTGS | 24x7 availability; instant settlement; no bank-hours dependency; correct for small monthly payout amounts |
| UPI Intent for data owner deposits | UPI Collect | UPI Collect is deprecated Feb 2026; Intent is the NPCI-mandated replacement; no change to UX concept, just underlying mechanism |
| Idempotency key from escrow_events table | Razorpay-generated key | Single source of truth; the escrow_events `idempotency_key` serves as the Razorpay `X-Payout-Idempotency` header directly, eliminating a mapping layer |

---

## Breaks in Our Case

- **ADR-024 assumed Razorpay Escrow+ supports programmable partial hold-and-release** ≠ **Razorpay Escrow+ is a regulated NBFC-only product with trustee approval required for every payout**
→ The escrow hold-and-release mechanism must be implemented as a custom internal ledger (the escrow_events table, ADR-016) with Razorpay Route as the transfer rail. The hold logic (30-day window, release multiplier, dual-window flag, seizure) lives entirely in the microservice. Razorpay is called only when a release or payout is computed. No change to ADR-024's design; only the implementation path changes.

- **UPI Collect flow deprecation 28 February 2026** ≠ **V2 uses UPI for data owner deposit collection**
→ The data owner deposit UI must use UPI Intent (app-based) or UPI QR code, not Collect (VPA entry). This affects the frontend checkout component for escrow deposit. Smart Collect 2.0 for the receiving side (assigning a virtual UPI ID per contract) is unaffected — it does not use the Collect flow. The backend deposit reconciliation is unchanged.

- **Route Linked Account cooling period is 24 hours** ≠ **providers should be able to receive transfers immediately after vetting**
→ Linked Accounts must be created at least 24 hours before the first transfer. In practice, this maps to: provider onboarding completes → Linked Account created → 24h later, chunk assignment begins. The onboarding pipeline must account for this gap. During the vetting period (4–6 months), this is not a constraint since the first transfer is weeks away.

- **Route settlement hold released on "next business day" after on_hold_until timestamp** ≠ **ADR-024 specifies release on the first of each month**
→ The microservice sets `on_hold_until` to the Unix timestamp of midnight on the last day of the current month. Razorpay releases on the next business day after that timestamp, which may be 2nd or 3rd of the following month if 1st falls on a weekend or bank holiday. The release window in ADR-024 ("first of each month") should be loosened to "within the first 3 business days of each month" to account for this. No change to the logic; only the disclosure to providers needs adjustment.

- **Route does not natively support percentage-based transfers — only fixed paise amounts** ≠ **ADR-024's release multiplier produces a percentage of holdings**
→ The microservice must compute the paise amount to release (escrow_held × release_multiplier) and pass the computed integer to the Transfer API. This is a microservice computation, not a Razorpay feature. All amounts must be in integer paise, consistent with ADR-016's `amount_paise BIGINT` column.

---

## Decisions Influenced

- **ADR-011 [#13 Fiat Escrow]** `CONFIRMED AND IMPLEMENTATION PATH CLARIFIED`
Razorpay is the correct payment gateway for V2. The product stack is: Smart Collect 2.0 (data owner deposit collection via virtual UPI IDs), Razorpay Route (transfer to provider Linked Accounts with settlement hold for escrow), and RazorpayX Payout API (monthly release to provider bank accounts). Razorpay Escrow+ is not used — it is a regulated NBFC product requiring trustee involvement that Vyomanaut does not qualify for. The PaymentProvider interface (ADR-011) maps to: `initiateEscrow` → Smart Collect 2.0 VPA assignment; `releaseEscrow` → Route transfer with `on_hold_until` set to month-end; `penalise` → Route settlement hold flag; `getBalance` → escrow_events table query (internal, not Razorpay).
*Because:* RazorpayX Escrow+ requires NBFC registration and trustee approval per payout. Route's `on_hold` / `on_hold_until` / Modify Settlement Hold API provides the programmable hold-and-release primitive ADR-024 requires without regulatory preconditions.

- **ADR-024 [#18 Economic Mechanism]** `IMPLEMENTATION CONSTRAINT ADDED`
The escrow hold-and-release mechanism is confirmed to be a custom internal ledger (escrow_events, ADR-016) with Razorpay Route as the outgoing transfer rail. The Route `on_hold: true` flag holds each transfer in the master account; the Modify Settlement Hold API releases it when the monthly release multiplier computation completes. The seizure mechanism must be triggered before settlement (i.e. the microservice must keep all transfers on hold until release is authorised) — Route reversal after settlement requires the provider's Linked Account to have positive balance, which cannot be guaranteed. The release window "first of each month" in ADR-024 should be updated to "within the first 3 business days of each month" to accommodate Route's next-business-day release timing.
*Because:* Route's `on_hold_until` releases on the next business day, not on the exact timestamp date.

- **ADR-012 [#13 Payment per Audit Passed]** `IMPLEMENTATION CONSTRAINT ADDED`
The payout idempotency key (`X-Payout-Idempotency` header) is mandatory for all Razorpay payout calls as of 15 March 2025. The escrow_events `idempotency_key VARCHAR(64)` column (ADR-016) must be passed directly as this header. The key format must be at least 10 characters, containing only alphanumerics, hyphens, and underscores. SHA256(provider_id + audit_period) satisfies this (64 hex characters). No schema change to ADR-016 or ADR-017 is required; only the payout service implementation must extract and forward the key.
*Because:* Razorpay enforces idempotency key at the API gateway level; missing header = rejected payout.

- **ADR-016 [#13 Payment DB Schema]** `CONFIRMED`
The escrow_events append-only table design (ADR-016) is correct given that Razorpay holds no escrow state — all hold logic is internal. The `amount_paise BIGINT` field is correct (Razorpay API accepts and returns paise as integers; no float arithmetic at any layer). The `idempotency_key` field serves double duty as the Razorpay `X-Payout-Idempotency` header value.
*Because:* Razorpay Route does not provide a query API for "how much have I paid provider X in the last 30 days" — the microservice must maintain that ledger itself.

---

## Disagreements

- **ADR-011's assumption that "UPI has zero per-transaction fee":** This is true for person-to-person UPI transfers through UPI apps. For programmatic payout via the Razorpay API to a provider's UPI VPA, Razorpay charges a fee (typically ₹2.5–₹5 for IMPS, variable for UPI-to-bank-account). The zero-fee property applies to the UPI rail at NPCI level, but Razorpay adds a service fee on top. The payout fee is deducted from the Razorpay master account balance, not from the provider payout, so provider earnings are unaffected. The microservice must maintain sufficient master account balance to cover both payouts and fees. The operational cost model should include estimated payout fees.
*Implication for us:* The storage rate (paise/GB/month) must be set high enough to cover Razorpay payout fees per provider per month, in addition to Vyomanaut's own margin.

---

## Open Questions

See open-questions.md — questions Q35-1 and Q35-2.