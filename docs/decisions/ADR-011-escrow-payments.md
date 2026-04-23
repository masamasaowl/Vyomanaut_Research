# ADR-011 — Fiat Escrow via UPI/Razorpay; India-First at Launch

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (provider)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 07, 33, 35

---

## Context

All surveyed DSNs (Filecoin, Storj token, Swarm SWAP) use cryptocurrency for payments. Every one of them introduced friction that hurt adoption or failed in production. The system needs a payment method that works for Indian providers and data owners without requiring crypto wallets.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Cryptocurrency (Storj token model) | Trustless; globally accessible | Price volatility; high onboarding friction; failed in production (Swarm) |
| Stripe Connect | Global; mature | Per-transaction fee; days to settle; requires Stripe account |
| **UPI via Razorpay or Cashfree** | Zero per-transaction fee; settles in seconds; universally available in India; no account friction | India-only; requires abstraction layer for future expansion |

## Decision

Fiat escrow exclusively at V2 launch. Payment via UPI using Razorpay or Cashfree as the payment gateway.

**Why India-first:**
- Ensures geographic proximity among providers, reducing transfer latency
- No currency conversion needed — all payments in INR
- UPI has zero per-transaction merchant fee
- UPI settles in seconds; Stripe Connect can take days
- UPI is available to every Indian with a bank account; Stripe requires additional account setup

**Abstraction layer (required from day one):**
The payment system is built behind a `PaymentProvider` interface:
```
initiateEscrow(data_owner_id, amount, contract_id)
releaseEscrow(provider_id, amount, audit_period)
penalise(provider_id, amount, reason)
getBalance(entity_id)
```
This allows Stripe or other gateways to be added without rewriting the payment logic.

Implementation mapping (from [Paper 35](../research/paper-35-razorpay-upi-docs.md) — Razorpay API documentation): initiateEscrow → Smart Collect 2.0 VPA assignment; releaseEscrow → Route transfer with on_hold_until set to month-end; penalise → Route settlement hold flag; getBalance → escrow_events table query (internal). Razorpay Escrow+ (the native escrow product) is not used — it requires NBFC registration and trustee approval that Vyomanaut does not qualify for. Route's programmable hold-and-release API provides the same primitive without regulatory preconditions.

Production precedent: Storj's held-earnings vetting model ([Paper 05](../research/paper-05-storj.md), Section 4.16) is the closest deployed equivalent. Storj holds a fraction of earnings for nine months; if a provider departs prematurely, the Satellite reclaims held payments to cover repair costs. Vyomanaut's 30-day rolling escrow hold and seizure on silent departure ([ADR-024](./ADR-024-economic-mechanism.md)) implement the same principle at a shorter hold window calibrated for Indian desktop providers.

**Escrow flow:**
Data owner deposits funds into escrow at contract start. The escrow holds funds until audit results release them to providers. The payment microservice handles all releases with idempotency keys to prevent double-payment (see [ADR-016](ADR-016-payment-db-schema.md)).

**Blockchain replacement:** 
Blockchain in all surveyed DSNs performs three separable functions ([Paper 07](../research/paper-07-sok-dsn.md)): (1) immutable audit log — replaced by append-only INSERT-only Postgres ([ADR-015](./ADR-015-audit-trail.md), [ADR-016](../research/paper-16-aont-rs-dispersal.md)); (2) automatic payment trigger on verified proof — replaced by the PaymentProvider.releaseEscrow call on audit pass ([ADR-012](./ADR-012-payment-basis.md)); (3) public dispute resolution — deferred to V3 Transparent Merkle Log ([ADR-015](./ADR-015-audit-trail.md)). The PaymentProvider interface is explicitly designed to replicate these three functions.

## Consequences

**Positive:**
- Zero per-transaction fee makes micro-payments economically viable
- Instant UPI settlement reduces provider cash flow friction
- Abstraction layer future-proofs the system for international expansion

**Negative / trade-offs:**
- India-only at launch limits the geographic scope of the provider network
- The payment microservice becomes a critical dependency — if it is down, no escrow releases can occur

**Open constraints:**
- International expansion requires the PaymentProvider interface to be implemented for a new gateway (Stripe Connect is the likely first candidate)

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): held-earnings vetting model (9-month hold, seizure on premature departure); Neuman proxy protocol for fraud-resistant bandwidth accounting
- [Paper 07 — SoK DSN](../research/paper-07-sok-dsn.md): three blockchain functions Vyomanaut must replicate; Swarm SWAP failure confirms payment-layer / transfer-layer decoupling is required
- [Paper 33 — Ihle et al.](../research/paper-33-ihle-incentive-mechanisms.md): Table 3 confirms centrally-managed currency is the production pattern for deterministic P2P storage incentives; Razorpay/UPI maps to Gramaglia's bank-managed model without jurisdiction restrictions
- [Paper 35 — Razorpay API Docs](../research/paper-35-razorpay-upi-docs.md): Escrow+ not applicable; Route + on_hold is the correct escrow primitive; UPI Collect deprecated Feb 2026 → use UPI Intent; idempotency key mandatory from 15 Mar 2025
