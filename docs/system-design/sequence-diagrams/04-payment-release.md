# Vyomanaut V2 — Payment Release Sequence Diagram

**Document ID:** `VYOM-SEQ-004`
**Version:** 1.0
**Status:** Authoritative
**Date:** April 2026
**Author:** Vyomanaut Engineering
**Repository:** [masamasaowl/Vyomanaut_Research](https://github.com/masamasaowl/Vyomanaut_Research)
**Companion documents:**
- [architecture.md §17 Payment System](../architecture.md#17-payment-system)
- [architecture.md §22 Runtime Flows — Flow 4](../architecture.md#22-runtime-flows)
- [requirements.md §6.10 Payment System](../requirements.md#610-payment-system)
- [ADR-011](../../decisions/ADR-011-escrow-payments.md) · [ADR-012](../../decisions/ADR-012-payment-basis.md) · [ADR-016](../../decisions/ADR-016-payment-db-schema.md) · [ADR-024](../../decisions/ADR-024-economic-mechanism.md)

---

## Overview

This diagram covers the four payment flows: (1) the monthly release computation that
converts audit pass history into actual bank transfers for providers; (2) the data
owner escrow deposit via UPI Intent; (3) escrow seizure when a provider departs
silently; and (4) idempotency and failure handling for Razorpay payout errors. The
primary correctness properties are: (1) all amounts are integer paise — floating-point
arithmetic is prohibited anywhere in the payment path; (2) the `X-Payout-Idempotency`
header prevents double-payment on retry; (3) withheld amounts from partial releases
roll forward rather than being seized. These properties derive from
[ADR-016](../../decisions/ADR-016-payment-db-schema.md), [ADR-012](../../decisions/ADR-012-payment-basis.md),
and [ADR-024](../../decisions/ADR-024-economic-mechanism.md).

---

## Participants

| Participant label | Role in this flow | Described in |
|---|---|---|
| `Microservice` | Monthly release computation; seizure orchestration; webhook handler | [architecture.md §18](../architecture.md#18-coordination-microservice) |
| `PostgreSQL` | `escrow_events`, `mv_provider_scores`, `audit_periods`, `providers` | [architecture.md §6](../architecture.md#6-component-overview) |
| `RazorpayRoute` | Programmable hold/release for provider earnings | [architecture.md §17](../architecture.md#17-payment-system) |
| `RazorpaySmartCollect` | Inbound UPI virtual account for data owner deposits | [architecture.md §17](../architecture.md#17-payment-system) |
| `RazorpayX` | Outbound bank transfer to provider's linked account | [architecture.md §17](../architecture.md#17-payment-system) |
| `DataOwnerClient` | Initiates escrow deposits via UPI Intent | [architecture.md §5](../architecture.md#5-system-context) |
| `ProviderBank` | Provider's UPI-linked bank account (terminal destination) | External |

---

## Happy Path 1 — Monthly Release Computation (23rd of Month)

On the 23rd of each month, the microservice computes each active provider's releasable
balance. The release multiplier is derived from the 30-day reliability score. A
dual-window deterioration flag catches providers who are degrading before the 72-hour
departure threshold fires. Razorpay releases on the next business day after
`on_hold_until` — the target window for providers is within the first 3 business days
of the following month. This gap is annotated explicitly to resolve the timing ambiguity
that exists across several prior documents.

```mermaid
sequenceDiagram
    %% Payment Release — Monthly release computation (23rd of month)
    %% ADR-024 §3 (release multiplier), ADR-016 (integer paise, idempotency),
    %% ADR-012 (per audit passed), FR-048, FR-049, FR-050

    participant MS  as Microservice
    participant PG  as PostgreSQL
    participant RR  as RazorpayRoute
    participant RX  as RazorpayX
    participant PB  as ProviderBank

    Note over MS: ── 23rd of MONTH, monthly release job ──<br/>SELECT provider_id FROM providers<br/>WHERE status IN ('ACTIVE', 'VETTING')<br/>AND frozen = false<br/>(Frozen providers skipped — seizure<br/>already in progress, ADR-024 §5)

    loop For each active provider (batched, ≤3.5/sec to stay under Razorpay rate limit)

        %% ── Step 1: Fetch 30-day and 7-day scores ───────────────────────────
        MS->>PG: SELECT score_30d, score_7d<br/>FROM mv_provider_scores<br/>WHERE provider_id = ?
        PG-->>MS: { score_30d: 0.92, score_7d: 0.71 }

        %% ── Step 2: Dual-window deterioration check ─────────────────────────
        Note over MS: Dual-window check (ADR-024 §3, FR-050):<br/>  score_30d − score_7d = 0.92 − 0.71 = 0.21<br/>  0.21 > 0.20 → dual_window_flag = TRUE<br/><br/>  effective_score = min(score_30d, score_7d)<br/>                  = min(0.92, 0.71) = 0.71<br/><br/>  This catches providers degrading before<br/>  the 72h departure threshold fires —<br/>  the PeerTrust dual-window signal<br/>  (Paper 31, ADR-024 §3).

        %% ── Step 3: Apply release multiplier ────────────────────────────────
        Note over MS: Release multiplier table (ADR-024 §3):<br/>  score ≥ 0.95 → 1.00 (full release)<br/>  score 0.80–0.94 → 0.75<br/>  score 0.65–0.79 → 0.50  ← applies here<br/>  score < 0.65 → 0.00 (hold in full)<br/><br/>  effective_score = 0.71<br/>  → multiplier = 0.50<br/><br/>  Withheld portion (0.50) is NOT seized —<br/>  it rolls forward into the next month's<br/>  escrow window. Seizure only occurs<br/>  on silent departure (ADR-024).

        %% ── Step 4: Compute releasable amount in paise ──────────────────────
        MS->>PG: SELECT SUM(amount_paise)<br/>FROM escrow_events<br/>WHERE provider_id = ?<br/>AND event_type = 'DEPOSIT'<br/>AND created_at > NOW() - INTERVAL '30 days'
        PG-->>MS: window_balance_paise = 75_000
        Note over MS: releasable_paise =<br/>  floor(75_000 × 0.50) = 37_500<br/>  (₹375 — all arithmetic in integer paise,<br/>  no FLOAT anywhere, ADR-016, NFR-046)<br/><br/>  idempotency_key =<br/>    SHA256(provider_id || audit_period_id)<br/>  as 64 hex chars (ADR-012, Paper 35).

        %% ── Step 5: Set Razorpay on_hold_until ──────────────────────────────
        Note over MS: on_hold_until = last_working_day_of_current_month()<br/>  Accounts for RBI bank holidays via static<br/>  lookup table updated each December.<br/>  (ADR-024, NFR-031, Paper 35)<br/><br/>  ⚠ TIMING CLARIFICATION (IC-008):<br/>  The microservice sets on_hold_until on the<br/>  23rd. Razorpay releases on the NEXT<br/>  BUSINESS DAY after on_hold_until —<br/>  not ON that date. Target window:<br/>  within the first 3 business days<br/>  of the FOLLOWING month.

        MS->>RR: PATCH /transfers/:transfer_id<br/>{ on_hold: true,<br/>  on_hold_until: "last_working_day_of_month",<br/>  X-Payout-Idempotency: idempotency_key }
        Note over RR: Razorpay stores the hold update.<br/>Will release the transfer on the<br/>next business day after on_hold_until.<br/>Razorpay Escrow+ NOT used —<br/>requires NBFC registration Vyomanaut<br/>does not have (ADR-011, Paper 35).
        RR-->>MS: 200 OK { transfer_id, on_hold_until }

        %% ── Step 6: Record the RELEASE event ────────────────────────────────
        MS->>PG: INSERT INTO escrow_events<br/>{ event_type='RELEASE',<br/>  amount_paise=37_500,<br/>  provider_id=?,<br/>  audit_period_id=?,<br/>  idempotency_key=SHA256(provider_id||period) }
        Note over PG: INSERT-only (Invariant 2). Balance<br/>is always recomputed as<br/>SUM(DEPOSIT) − SUM(RELEASE+SEIZURE).<br/>No mutable balance column exists. (ADR-016)
        PG-->>MS: OK

    end

    %% ── Razorpay releases on schedule ────────────────────────────────────
    Note over RR: ── NEXT BUSINESS DAY AFTER on_hold_until ──<br/>Razorpay Route releases the transfer.<br/>Funds move to RazorpayX Payout.

    RR->>RX: Initiate payout to provider's Linked Account<br/>{ amount: 37_500 paise, account: "acc_XXXX" }
    RX->>PB: IMPS / UPI transfer<br/>₹375.00 → provider's bank account
    PB-->>RX: Settlement confirmed
    RX-->>RR: payout.processed webhook
    RR-->>MS: payout.processed webhook<br/>{ payout_id, amount, status: "processed" }

    Note over MS: Webhook received. No additional<br/>action needed — RELEASE event already<br/>recorded at INSERT time. The payout_id<br/>is logged for reconciliation only.
```

### Cross-reference: diagram steps to ADRs and requirements

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | Only non-frozen providers processed; frozen = seizure in progress | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §5 |
| 2 | Scores read from `mv_provider_scores` materialised view (up to 60 s stale — acceptable for monthly computation) | [ADR-008](../../decisions/ADR-008-reliability-scoring.md), [data-model.md §7](../data-model.md#7-materialised-views) |
| 3 | `score_30d − score_7d > 0.20` → `dual_window_flag = TRUE` → use lower score | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §3, [FR-050](../requirements.md#610-payment-system) |
| 4 | Release multiplier table: 0.50 band for score 0.65–0.79 | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §3, [FR-049](../requirements.md#610-payment-system) |
| 5 | Withheld portion rolls forward — not seized until silent departure | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), trade-offs.md #19 |
| 6 | `floor()` used for paise arithmetic; all amounts integer paise, no float | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), [NFR-046](../requirements.md#77-compliance-and-payments) |
| 7 | `on_hold_until` = last working day of current month; RBI holiday table (NFR-031) | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), [NFR-031](../requirements.md#77-compliance-and-payments) |
| 8 | `X-Payout-Idempotency` header mandatory since 15 March 2025 | [ADR-012](../../decisions/ADR-012-payment-basis.md), [FR-047](../requirements.md#610-payment-system), Paper 35 |
| 9 | Razorpay Escrow+ NOT used; Route `on_hold` is the correct primitive | [ADR-011](../../decisions/ADR-011-escrow-payments.md), Paper 35 |
| 10 | `RELEASE` event inserted before Razorpay confirms; idempotency key prevents duplicate on retry | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), [NFR-022](../requirements.md#75-reliability-and-correctness) |

### What this diagram does not show

- How per-audit-pass DEPOSIT events accumulate throughout the month — each audit PASS generates a paise-denominated DEPOSIT row in `escrow_events`; this background accumulation is not shown to keep the monthly release diagram readable.
- The vetting period modifier (60-day hold, 50% release cap) — `providers.status = 'VETTING'` triggers modified hold window logic; same diagram flow, different parameters ([ADR-024](../../decisions/ADR-024-economic-mechanism.md) §6, [FR-051](../requirements.md#610-payment-system)).
- Razorpay Linked Account creation and 24-hour cooling period — covered in [05-provider-lifecycle.md](./05-provider-lifecycle.md).
- The storage rate (paise per GB per month) — a product decision (OQ-001); the diagram is parameterised on `storage_rate_paise_per_gb_per_month` which is injected at runtime.

---

## Happy Path 2 — Data Owner Escrow Deposit via UPI Intent

When a data owner wants to store files, they must first deposit escrow funds. The system
uses Razorpay Smart Collect 2.0 virtual UPI accounts. UPI Collect is explicitly not used —
it was deprecated by NPCI on 28 February 2026. The microservice does not credit the
escrow balance until the Razorpay webhook confirms the payment has settled.

```mermaid
sequenceDiagram
    %% Payment Release — Data owner escrow deposit via UPI Intent
    %% ADR-011, FR-006, NFR-029

    participant DO  as DataOwnerClient
    participant MS  as Microservice
    participant RSC as RazorpaySmartCollect
    participant PG  as PostgreSQL

    DO->>MS: POST /api/v1/owner/deposit<br/>{ amount_paise: 100_000 }
    Note over MS: Validate: amount_paise must be a<br/>positive integer (no float). (ADR-016)<br/>Retrieve smart_collect_vpa for this owner<br/>from owners.smart_collect_vpa.<br/>VPA is pre-provisioned at registration.

    MS-->>DO: 200 OK<br/>{ vpa: "vyomanaut.owner123@razorpay",<br/>  qr_code_url: "https://...",<br/>  expires_at: "2026-04-28T10:15:00Z" }

    Note over DO: Data owner opens their UPI app<br/>(GPay, PhonePe, Paytm, etc.) and pays<br/>to the displayed VPA or scans the QR code.<br/>This is the UPI INTENT flow —<br/>UPI Collect (push-based) is deprecated<br/>as of 2026-02-28 and must NOT be used.<br/>(NFR-029, Paper 35, ADR-011)

    DO->>RSC: UPI Intent payment: ₹1,000 → VPA

    Note over RSC: Razorpay Smart Collect 2.0 receives<br/>the UPI payment and reconciles it<br/>against the virtual account.

    RSC->>MS: POST /webhooks/razorpay<br/>virtual_account.payment.captured<br/>{ owner_id, amount_paise: 100_000,<br/>  payment_id: "pay_XXXXXXXX" }

    Note over MS: Webhook verified via Razorpay signature<br/>header. idempotency_key =<br/>SHA-256("deposit" || payment_id)<br/>to prevent duplicate crediting on<br/>webhook retry. (ADR-016)

    MS->>PG: INSERT INTO escrow_events<br/>{ provider_id: owner_escrow_account_id,<br/>  event_type='DEPOSIT',<br/>  amount_paise=100_000,<br/>  idempotency_key=SHA-256('deposit'||payment_id) }
    Note over PG: INSERT-only ledger. Balance =<br/>SUM(DEPOSIT) − SUM(RELEASE+SEIZURE).<br/>Data owner can now upload files up to<br/>cost_for_30_days(total_file_size) ≤ 100_000 paise. (FR-014)
    PG-->>MS: OK

    MS-->>DO: (next GET /owner/balance shows updated balance)
```

### Cross-reference

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | `amount_paise` validated as positive integer — no float accepted | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), [NFR-046](../requirements.md#77-compliance-and-payments) |
| 2 | UPI Intent flow (QR / app redirect) — UPI Collect deprecated 28 Feb 2026 | [ADR-011](../../decisions/ADR-011-escrow-payments.md), [NFR-029](../requirements.md#77-compliance-and-payments) |
| 3 | Escrow credited only on `virtual_account.payment.captured` webhook — not on client claim | [ADR-011](../../decisions/ADR-011-escrow-payments.md) |
| 4 | Idempotency key `SHA-256("deposit" || payment_id)` prevents double-credit on webhook retry | [ADR-016](../../decisions/ADR-016-payment-db-schema.md) |
| 5 | DEPOSIT event enables upload: `balance ≥ cost_for_30_days(file_size)` check | [FR-014](../requirements.md#62-data-owner--file-upload) |

---

## Happy Path 3 — Escrow Seizure on Silent Departure

When the departure detector declares a provider silently departed (triggered from the
repair flow in [03-repair-flow.md](./03-repair-flow.md)), the payment system seizes all
earnings in the 30-day rolling window into the repair reserve fund. If a Razorpay
transfer has already been initiated but not yet settled, a Route reversal is issued.

```mermaid
sequenceDiagram
    %% Payment Release — Escrow seizure on silent departure
    %% ADR-024 §5, ADR-007, FR-035

    participant MS  as Microservice
    participant PG  as PostgreSQL
    participant RR  as RazorpayRoute

    Note over MS: Called from repair flow (03-repair-flow.md)<br/>after status='DEPARTED' is set.<br/>Seizure is a coordinated operation —<br/>only the single payment microservice<br/>may execute it. (ADR-013, non-I-confluent)

    MS->>PG: UPDATE providers<br/>SET frozen = true<br/>WHERE provider_id = ?
    Note over PG: frozen=true blocks any further<br/>DEPOSIT events for this provider.<br/>Ongoing audits cease (chunk_assignments<br/>already hard-deleted in repair flow).
    PG-->>MS: OK

    MS->>PG: SELECT SUM(amount_paise)<br/>FROM escrow_events<br/>WHERE provider_id = ?<br/>AND created_at > NOW() - INTERVAL '30 days'<br/>AND event_type = 'DEPOSIT'
    PG-->>MS: rolling_30d_balance_paise = 60_000

    Note over MS: idempotency_key =<br/>SHA-256(provider_id || 'seizure' || departed_at)<br/>Prevents double-seizure if the job<br/>is accidentally re-run. (ADR-016)

    MS->>PG: INSERT INTO escrow_events<br/>{ event_type='SEIZURE',<br/>  amount_paise=60_000,<br/>  provider_id=?,<br/>  idempotency_key=SHA-256(provider_id||'seizure'||departed_at) }
    PG-->>MS: OK

    Note over MS: Check if any Razorpay transfer<br/>is in-flight (initiated but not yet<br/>settled to the provider's bank).<br/>If so: issue a Route reversal to<br/>reclaim those funds.

    MS->>RR: POST /transfers/:transfer_id/reversals<br/>{ description: "Provider departed silently" }
    Note over RR: Razorpay reverses the in-flight<br/>transfer. Funds return to the<br/>Vyomanaut master account.<br/>These returned funds top up the<br/>repair reserve. If transfer already<br/>settled to provider's bank: reversal<br/>cannot be issued — only the<br/>60_000 paise in escrow_events<br/>is seized (which represents the<br/>current 30-day rolling window).
    RR-->>MS: 200 OK { reversal_id }

    Note over MS: Seized funds cover:<br/>  Qpeek repair bandwidth cost<br/>  ≈ 793 GB at N=1,000 providers<br/>  (Giroire Formula 2, Paper 10).<br/>  At any provider MTTF > 180 days,<br/>  seized escrow exceeds repair cost.<br/>  (ADR-024 §5, capacity.md §3.2)
```

### Cross-reference

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | Seizure is a non-I-confluent operation — single payment service only | [ADR-013](../../decisions/ADR-013-consistency-model.md) |
| 2 | `frozen = true` blocks further deposits; prevents escrow state race | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §5 |
| 3 | `SEIZURE` idempotency key prevents double-seizure on job re-run | [ADR-016](../../decisions/ADR-016-payment-db-schema.md) |
| 4 | Razorpay Route reversal issued for in-flight transfers | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §5, [ADR-011](../../decisions/ADR-011-escrow-payments.md) |
| 5 | `FR-035`: seizure triggered automatically — no operator intervention needed | [FR-035](../requirements.md#67-provider--exit-and-departure) |

---

## Failure Path 1 — Razorpay Payout API Returns Error

If the Razorpay payout call fails (network error, temporary API unavailability, or
provider bank rejection), Razorpay automatically reverses the payout and fires a
webhook. The microservice records a `REVERSED` state and retries on the next monthly
cycle using the same idempotency key — preventing any double-payment.

```mermaid
sequenceDiagram
    %% Payment Release — Razorpay payout failure and reversal
    %% ADR-012, ADR-016, FR-047, Paper 35

    participant MS  as Microservice
    participant PG  as PostgreSQL
    participant RR  as RazorpayRoute
    participant RX  as RazorpayX

    Note over MS: Monthly release job fires (23rd of month).<br/>RELEASE event already INSERTed to escrow_events.<br/>Razorpay PATCH on_hold_until already sent.<br/>Now attempting the actual payout transfer.

    MS->>RX: POST /payouts<br/>{ account_number: "...",<br/>  amount: 37_500,<br/>  currency: "INR",<br/>  mode: "UPI",<br/>  X-Payout-Idempotency: idempotency_key }
    Note over RX: ⚠ Payout call fails — provider's<br/>UPI handle is invalid (bank account closed).

    RX-->>MS: 422 Unprocessable Entity<br/>{ error: "INVALID_UPI" }

    Note over MS: Record the failure. Do NOT insert<br/>a new escrow_events row — the RELEASE<br/>event from Step 6 of the release flow<br/>remains in the ledger as an ATTEMPTED<br/>state. The actual bank transfer did not<br/>complete. Note the failure in logs<br/>and alert the provider via daemon<br/>notification.

    Note over RX: Razorpay queues a reversal<br/>for the payout amount.

    RX->>MS: POST /webhooks/razorpay<br/>payout.reversed<br/>{ payout_id, amount: 37_500,<br/>  idempotency_key }

    Note over MS: Webhook received. The idempotency_key<br/>matches the original RELEASE event.<br/>Insert a REVERSED marker to the escrow<br/>ledger so the balance is correctly<br/>restored for next month's computation.

    MS->>PG: INSERT INTO escrow_events<br/>{ event_type='RELEASE',<br/>  amount_paise=-37_500,<br/>  idempotency_key=SHA-256(idempotency_key||'reversal') }
    Note over PG: A negative RELEASE effectively<br/>returns the amount to the provider's<br/>available balance. Alternative:<br/>model this as a separate event_type<br/>'REVERSAL' — team decision before<br/>schema is locked. Either approach<br/>preserves INSERT-only invariant. (ADR-016)
    PG-->>MS: OK

    Note over MS: On next monthly cycle (following 23rd):<br/>  Same idempotency_key is used for<br/>  the new PATCH on_hold_until call.<br/>  Razorpay returns the existing payout<br/>  state rather than creating a duplicate.<br/>  Provider must update their UPI handle<br/>  via the provider settings interface<br/>  before the retry succeeds. (FR-047)
```

### Cross-reference

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | `X-Payout-Idempotency` header mandatory on every payout call since 15 March 2025 | [FR-047](../requirements.md#610-payment-system), Paper 35 |
| 2 | Payout failure does NOT delete the RELEASE escrow row — it was recorded before the transfer | [ADR-016](../../decisions/ADR-016-payment-db-schema.md) |
| 3 | `payout.reversed` webhook triggers negative RELEASE to restore balance | [ADR-012](../../decisions/ADR-012-payment-basis.md), Paper 35 |
| 4 | Retry on next monthly cycle uses same idempotency key — no duplicate transfer | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), [FR-047](../requirements.md#610-payment-system) |

---

## Failure Path 2 — Duplicate Monthly Release Job Run (Idempotency)

If the monthly release job is accidentally triggered twice (e.g., a scheduler bug or
operator re-run), the idempotency key on `escrow_events` and the Razorpay
`X-Payout-Idempotency` header together prevent any double payment.

```mermaid
sequenceDiagram
    %% Payment Release — Idempotent duplicate job run
    %% ADR-016 (idempotency_key UNIQUE), NFR-030

    participant MS  as Microservice
    participant PG  as PostgreSQL
    participant RR  as RazorpayRoute

    Note over MS: Monthly release job accidentally run twice<br/>for provider P in audit_period AP.

    MS->>RR: PATCH /transfers/:transfer_id<br/>{ on_hold_until: ...,<br/>  X-Payout-Idempotency: SHA-256(provider_id||AP) }
    Note over RR: Razorpay detects same idempotency key<br/>from earlier call. Returns existing<br/>transfer state without creating a<br/>second hold update.
    RR-->>MS: 200 OK { existing transfer state }

    MS->>PG: INSERT INTO escrow_events<br/>{ idempotency_key: SHA-256(provider_id||AP), ... }
    Note over PG: UNIQUE constraint on idempotency_key<br/>fires. INSERT is rejected.<br/>No duplicate RELEASE row created.<br/>Balance unchanged. (ADR-016)
    PG-->>MS: 409 Conflict — duplicate key violation

    Note over MS: Job detects 409 on escrow_events INSERT.<br/>Treats as idempotent success —<br/>the RELEASE was already recorded.<br/>release_computed flag in audit_periods<br/>is already TRUE. Job exits cleanly.<br/>Provider receives exactly one payment.<br/>(FR-047, NFR-030)
```

---

## Invariants Demonstrated

| Invariant | Where it appears in this flow | Source |
|---|---|---|
| All amounts are integer paise — no float | Happy Path 1 step 6: `floor(75_000 × 0.50) = 37_500`; explicitly annotated | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), Invariant 4 in [data-model.md](../data-model.md#3-design-invariants) |
| `escrow_events` is INSERT-only | Happy Path 1 step 10: RELEASE row inserted; Failure Path 1: REVERSED row inserted; no UPDATE or DELETE appears | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), Invariant 2 in [data-model.md](../data-model.md#3-design-invariants) |
| Idempotency key prevents double-payment | Failure Path 2: 409 on duplicate `idempotency_key` → job exits cleanly | [ADR-016](../../decisions/ADR-016-payment-db-schema.md) |
| Razorpay releases on next business day after `on_hold_until`, not on the date itself | Happy Path 1: timing clarification annotation explicitly states "next business day after" | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), Paper 35 |
| Withheld partial-release amounts roll forward — not seized | Happy Path 1 step 5: withheld 0.50 fraction annotated "NOT seized — rolls forward" | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) |
| Seizure is a non-I-confluent coordinated operation | Happy Path 3 step 1: "only the single payment microservice may execute it" | [ADR-013](../../decisions/ADR-013-consistency-model.md) |

---

## Related Diagrams

- **[03-repair-flow.md](./03-repair-flow.md)** — the escrow seizure in Happy Path 3 of this diagram is triggered by the departure detection sequence in that flow; repair job enqueuing precedes the seizure call.
- **[02-audit-cycle.md](./02-audit-cycle.md)** — each PASS result in the audit cycle generates a per-audit paise DEPOSIT event to the provider's `escrow_events`; the monthly accumulation of those events is what Happy Path 1 releases.
- **[05-provider-lifecycle.md](./05-provider-lifecycle.md)** — the Razorpay Linked Account creation and 24-hour cooling period that gate payment eligibility are shown in the registration sequence of that diagram.